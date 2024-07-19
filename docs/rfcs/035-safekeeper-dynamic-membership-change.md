# Safekeeper dynamic membership change

To quickly recover from safekeeper node failures and do rebalancing we need to
be able to change set of safekeepers the timeline resides on. The procedure must
be safe (not lose committed log) regardless of safekeepers and compute state. It
should be able to progress if any majority of old safekeeper set, any majority
of new safekeeper set and compute are up and connected. This is known as a
consensus membership change. It always involves two phases: 1) switch old
majority to old + new configuration, preventing commits without acknowledge from
the new set 2) bootstrap the new set by ensuring majority of the new set has all
data which ever could have been committed before the first phase completed;
after that switch is safe to finish. Without two phases switch to the new set
which quorum might not intersect with quorum of the old set (and typical case of
ABC -> ABD switch is an example of that, because quorums AC and BD don't
intersect). Furthermore, procedure is typically carried out by the consensus
leader, and so enumeration of configurations which establishes order between
them is done through consensus log.

In our case consensus leader is compute (walproposer), and we don't want to wake
up all computes for the change. Neither we want to fully reimplement the leader
logic second time outside compute. Because of that the proposed algorithm relies
for issuing configurations on the external fault tolerant (distributed) strongly
consisent storage with simple API: CAS (compare-and-swap) on the single key.
Properly configured postgres suits this.

In the system consensus is implemented at the timeline level, so algorithm below
applies to the single timeline.

## Algorithm

### Definitions

A configuration is

```
struct Configuration {
    generation: Generation, // a number uniquely identifying configuration
    sk_set: Vec<NodeId>, // current safekeeper set
    new_sk_set: Optional<Vec<SafekeeperId>>,
}
```

Configuration with `new_set` present is used for the intermediate step during
the change and called joint configuration. Generations establish order of
generations: we say `c1` is higher than `c2` if `c1.generation` >
`c2.generation`.

### Persistently stored data changes

Safekeeper starts storing its current configuration in the control file. Update
of is atomic, so in-memory value always matches the persistent one.

External CAS providing storage (let's call it configuration storage here) also
stores configuration for each timeline. It is initialized with generation 1 and
initial set of safekeepers during timeline creation. Executed CAS on it must
never be lost.

### Compute <-> safekeeper protocol changes

`ProposerGreeting` message carries walproposer's configuration if it is already
established (see below), else null.  `AcceptorGreeting` message carries
safekeeper's current `Configuration`. All further messages (`VoteRequest`,
`VoteResponse`, `ProposerElected`, `AppendRequest`, `AppendResponse`) carry
generation number, of walproposer in case of wp->sk message or of safekeeper in
case of sk->wp message.

### Safekeeper changes

Basic rule: once safekeeper observes configuration higher than his own it
immediately switches to it. It must refuse all messages with lower generation
that his. It also refuses messages if it is not member of the current
generation, though it is likely not unsafe to process them (walproposer should
ignore them anyway).

If there is non null configuration in `ProposerGreeting` and it is higher than
current safekeeper one, safekeeper switches to it.

Safekeeper sends its current configuration in its first message to walproposer
`AcceptorGreeting`. It refuses all other walproposer messages if the
configuration generation in them is less than its current one. Namely, it
refuses to vote, to truncate WAL in `handle_elected` and to accept WAL. In
response it sends its current configuration generation to let walproposer know.

Safekeeper gets `PUT /v1/tenants/{tenant_id}/timelines/{timeline_id}/configuration` 
accepting `Configuration`. Safekeeper switches to the given conf it is higher than its
current one and ignores it otherwise. In any case it replies with
```
struct ConfigurationSwitchResponse {
    conf: Configuration,
    last_log_term: Term,
    flush_lsn: Lsn,
    term: Term, // not used by this RFC, but might be useful for observability
}
```

### Compute (walproposer) changes

Basic rule is that joint configuration requires votes from majorities in the
both `set` and `new_sk_set`.

Compute receives list of safekeepers to connect to from the control plane as
currently and tries to communicate with all of them. However, the list does not
define consensus members. Instead, on start walproposer tracks highest
configuration it receives from `AcceptorGreeting`s. Once it assembles greetings
from majority of `sk_set` and majority of `new_sk_set` (if it is present), it
establishes this configuration as its own and moves to voting. 

It should stop talking to safekeepers not listed in the configuration at this
point, though it is not unsafe to continue doing so.

To be elected it must receive votes from both majorites if `new_sk_set` is present.
Similarly, to commit WAL it must receive flush acknowledge from both majorities.

If walproposer hears from safekeeper configuration higher than his own (i.e.
refusal to accept due to configuration change) it simply restarts.

### Change algorithm

The following algorithm can be executed anywhere having access to configuration
storage and safekeepers. It is safe to interrupt / restart it and run multiple
instances of it concurrently, though likely one of them won't make
progress then. It accepts `desired_set: Vec<NodeId>` as input. 

Algorithm will refuse to make the change if it encounters previous interrupted
change attempt, but in this case it will try to finish it.

It will eventually converge if old majority, new majority and configuration
storage are reachable.

1) Fetch current timeline configuration from the configuration storage.
2) If it is already joint one and `new_set` is different from `desired_set`
   refuse to change. However, assign join conf to (in memory) var
   `join_conf` and proceed to step 4 to finish the ongoing change.
3) Else, create joint `joint_conf: Configuration`: increment current conf number
   `n` and put `desired_set` to `new_sk_set`. Persist it in the configuration
   storage by doing CAS on the current generation: change happens only if
   current configuration number is still `n`. Apart from guaranteeing uniqueness
   of configurations, CAS linearizes them, ensuring that new configuration is
   created only following the previous one when we know that the transition is
   safe. Failed CAS aborts the procedure.
4) Call `PUT` `configuration` on safekeepers from the current set,
   delivering them `joint_conf`. Collecting responses from majority is required
   to proceed. If any response returned generation higher than 
   `joint_conf.generation`, abort (another switch raced us). Otherwise, choose
   max `<last_log_term, flush_lsn>` among responses and establish it as 
   (in memory) `sync_position`. We can't finish switch until majority 
   of the new set catches up to this position because data before it 
   could be committed without ack from the new set.
4) Initialize timeline on safekeeper(s) from `new_sk_set` where it 
   doesn't exist yet by doing `pull_timeline` from the majority of the 
   current set. Doing  that on majority of `new_sk_set` is enough to
   proceed, but it is reasonable to ensure that all `new_sk_set` members
   are initialized -- if some of them are down why are we migrating there?
5) Call `PUT` `configuration` on safekeepers from the new set,
   delivering them `joint_conf` and collecting their positions. This will
   switch them to the `joint_conf` which generally won't be needed 
   because `pull_timeline` already includes it and plus additionally would be
   broadcast by compute. More importantly, we may proceed to the next step 
   only when `<last_log_term, flush_lsn>` on the majority of the new set reached 
   `sync_position`. Similarly, on the happy path this is not needed because 
   `pull_timeline` already includes it. However, it is better to double
    check to be safe. For example, timeline could have been created earlier e.g.
    manually or after try-to-migrate, abort, try-to-migrate-again sequence.   
6) Create `new_conf: Configuration` incrementing `join_conf` generation and having new 
   safekeeper set as `sk_set` and None `new_sk_set`. Write it to configuration 
   storage under one more CAS.
7) Call `PUT` `configuration` on safekeepers from the new set,
   delivering them `new_conf`. It is enough to deliver it to the majority 
   of the new set; the rest can be updated by compute.

I haven't put huge effort to make the description above very precise, because it
is natural language prone to interpretations anyway. Instead I'd like to make TLA+
spec of it.

Description above focuses on safety. To make the flow practical and live, here a few more 
considerations.
1) It makes sense to ping new set to ensure it we are migrating to live node(s) before 
  step 3.
2) If e.g. accidentally wrong new sk set has been specified, before CAS in step `6` is completed 
   it is safe to rollback to the old conf with one more CAS.
3) On step 4 timeline might be already created on members of the new set for various reasons; 
   the simplest is the procedure restart. There are more complicated scenarious like mentioned
   in step 5. Deleting and re-doing `pull_timeline` is generally unsafe without involving 
   generations, so seems simpler to treat existing timeline as success. However, this also 
   has a disadvantage: you might imagine an surpassingly unlikely schedule where condition in
   the step 5 is never reached until compute is (re)awaken up to synchronize new member(s).
   I don't think we'll observe this in practice, but can add waking up compute if needed.
4) In the end timeline should be locally deleted on the safekeeper(s) which are
   in the old set but not in the new one, unless they are unreachable. To be
   safe this also should be done under generation number.
5) If current conf fetched on step 1 is already not joint and members equal to `desired_set`,
   jump to step 7, using it as `new_conf`.

## Implementation

The procedure ought to be driven from somewhere. Obvious candidates are control
plane and storage_controller; and as each of them already has db we don't want
yet another storage. I propose to manage safekeepers in storage_controller
because 1) since it is in rust it simplifies simulation testing (more on this
below) 2) it already manages pageservers. 

This assumes that migration will be fully usable only after we migrate all
tenants/timelines to storage_controller. It is discussible whether we want also
to manage pageserver attachments for all of these, but likely we do.

This requires us to define

### storage_controller <-> control plane interface

First of all, control plane should
[change](https://neondb.slack.com/archives/C03438W3FLZ/p1719226543199829)
storing safekeepers per timeline instead of per tenant because we can't migrate
tenants atomically. 

The important question is how updated configuration is delivered from
storage_controller to control plane to provide it to computes. As always, there
are two options, pull and push. Let's do it the same push as with pageserver
`/notify-attach` because 1) it keeps storage_controller out of critical compute
start path 2) provides easier upgrade: there won't be such a thing as 'timeline
managed by control plane / storcon', cplane just takes the value out of its db
when needed 3) uniformity. It makes storage_controller responsible for retrying notifying
control plane until it succeeds.

So, cplane `/notify-safekeepers` for the timeline accepts `Configuration` and
updates it in the db if the provided conf generation is higher (the cplane db
should also store generations for this). Similarly to [`/notify-attach`](https://www.notion.so/neondatabase/Storage-Controller-Control-Plane-interface-6de56dd310a043bfa5c2f5564fa98365), it
should update db which makes the call successful, and then try to schedule
`apply_config` if possible, it is ok if not. storage_controller 
should rate limit calling the endpoint, but likely this won't be needed, as migration
throughput is limited by `pull_timeline`.

Timeline (branch) creation in cplane should call storage_controller POST
`tenant/:tenant_id/timeline` like it currently does for sharded tenants.
Response should be augmented with `safekeeper_conf: Configuration`. The call
should be retried until succeeds.

Timeline deletion and tenant deletion in cplane should call appropriate
storage_controller endpoints like it currently does for sharded tenants. The
calls should be retried until they succeed.

### storage_controller implementation

Current 'load everything on startup and keep in memory' easy design is fine.
Single timeline shouldn't take more than 100 bytes (it's 16 byte tenant_id, 16
byte timeline_id, int generation, vec of ~3 safekeeper ids plus some flags), so
10^6 of timelines shouldn't take more than 100MB.

Similar to pageserver attachment Intents storage_controller would have in-memory
`MigrationRequest` (or its absense) for each timeline and pool of tasks trying
to make these request reality; this ensures one instance of storage_controller
won't do several migrations on the same timeline concurrently. In the first
version it is simpler to have more manual control and no retries, i.e. migration
failure removes the request. Later we can build retries and automatic
scheduling/migration. `MigrationRequest` is
```
enum MigrationRequest {
    To(Vec<NodeId>),
    FinishPending,
}
```

`FinishPending` requests to run the procedure to ensure state is clean: current
configuration is not joint and majority of safekeepers are aware of it, but do
not attempt to migrate anywhere. If current configuration fetched on step 1 is
not joint it jumps to step 7. It should be run at startup for all timelines (but
similarly, in the first version it is ok to trigger it manually).

#### Schema

`safekeepers` table mirroring current `nodes` should be added, except that for
`scheduling_policy` field (seems like `status` is a better name for it): it is enough
to have at least in the beginning only 3 fields: 1) `active` 2) `unavailable` 3)
`decomissioned`.

`timelines` table:
```
table! {
    // timeline_id is primary key
    timelines (timeline_id) {
        timeline_id -> Varchar,
        tenant_id -> Varchar,
        generation -> Int4,
        sk_set -> Jsonb, // list of safekeeper ids
        new_sk_set -> Nullable<Jsonb>, // list of safekeeper ids, null if not join conf
        cplane_notified_generation -> Int4,
    }
}
```

#### API

Node management is similar to pageserver:
1) POST `/control/v1/safekeepers` upserts safekeeper.
2) GET `/control/v1/safekeepers` lists safekeepers.
3) GET `/control/v1/safekeepers/:node_id` gets safekeeper.
4) PUT `/control/v1/safekepers/:node_id/status` changes status to e.g.
   `unavailable` or `decomissioned`. Initially it is simpler not to schedule any
    migrations here.

Safekeeper deploy scripts should register safekeeper at storage_contorller as
they currently do with cplane, under the same id.

Timeline creation/deletion: already existing POST `tenant/:tenant_id/timeline`
would 1) choose initial set of safekeepers; 2) write to the db initial
`Configuration` with `INSERT ON CONFLICT DO NOTHING` returning existing row in
case of conflict; 3) create timeline on the majority of safekeepers (already
created is ok).

We don't want to block timeline creation when one safekeeper is down. Currently
this is solved by compute implicitly creating timeline on any safekeeper it is
connected to. This creates ugly timeline state on safekeeper when timeline is
created, but start LSN is not defined yet. It would be nice to remove this; to
do that, controller can in the background retry to create timeline on
safekeeper(s) which missed that during initial creation call. It can do that
through `pull_timeline` from majority so it doesn't need to remember
`parent_lsn` in its db.

Timeline deletion removes the row from the db and forwards deletion to the
current configuration members. Without additional actions deletions might leak,
see below on this; initially let's ignore these, reporting to cplane success if
at least one safekeeper deleted the timeline (this will remove s3 data).

Tenant deletion repeats timeline deletion for all timelines.

Migration API: the first version is the simplest and the most imperative:
1) PUT `/control/v1/safekeepers/migrate` schedules `MigrationRequest`s to move
all timelines from one safekeeper to another. It accepts json
```
{
    "src_sk": u32,
    "dst_sk": u32,
    "limit": Optional<u32>,
}
```

Returns list of scheduled requests.

2) PUT `/control/v1/tenant/:tenant_id/timeline/:timeline_id/safekeeper_migrate` schedules `MigrationRequest`
   to move single timeline to given set of safekeepers:
{
    "desired_set": Vec<u32>,
}

Returns scheduled request.

Similar call should be added for the tenant.

It would be great to have some way of subscribing to the results (appart from
looking at logs/metrics).

Migration is executed as described above. One subtlety is that (local) deletion on
source safekeeper might fail, which is not a problem if we are going to
decomission the node but leaves garbage otherwise. I'd propose in the first version
1) Don't attempt deletion at all if node status is `unavailable`.
2) If it failed, just issue warning.
And add PUT `/control/v1/safekeepers/:node_id/scrub` endpoint which would find and 
remove garbage timelines for manual use. It will 1) list all timelines on the 
safekeeper 2) compare each one against configuration storage: if timeline 
doesn't exist at all (had been deleted), it can be deleted. Otherwise, it can 
be deleted under generation number if node is not member of current generation.

Automating this is untrivial; we'd need to register all potential missing
deletions <tenant_id, timeline_id, generation, node_id> in the same transaction
which switches configurations. Similarly when timeline is fully deleted to
prevent cplane operation from blocking when some safekeeper is not available
deletion should be also registered.

3) GET `/control/v1/tenant/:tenant_id/timeline/:timeline_id/` should return
   current in memory state of the timeline and pending `MigrationRequest`,
   if any.

4) PUT `/control/v1/tenant/:tenant_id/timeline/:timeline_id/safekeeper_migrate_abort` tries to abort the
   migration by switching configuration from the joint to the one with (previous) `sk_set` under CAS
   (incrementing generation as always).

#### Dealing with multiple instances of storage_controller

Operations described above executed concurrently might create some errors but do
not prevent progress, so while we normally don't want to run multiple instances
of storage_controller it is fine to have it temporarily, e.g. during redeploy.

Any interactions with db update in-memory controller state, e.g. if migration
request failed because different one is in progress, controller remembers that
and tries to finish it.

## Testing

`neon_local` should be switched to use storage_controller, playing role of
control plane.

There should be following layers of tests:
1) Model checked TLA+ spec specifies the algorithm and verifies its basic safety.

2) To cover real code and at the same time test many schedules we should have
   simulation tests. For that, configuration storage, storage_controller <->
   safekeeper communication and pull_timeline need to be mocked and main switch
   procedure wrapped to as a node (thread) in simulation tests, using these
   mocks. Test would inject migrations like it currently injects
   safekeeper/walproposer restars. Main assert is the same -- committed WAL must
   not be lost.

3) Additionally it would be good to have basic tests covering the whole system.

## Order of implementation and rollout

note that 
- core can be developed ignoring cplane integration (neon_local will use storcon, but prod not)
- there is a lot of infra work and it woud be great to separate its rollout from the core
- wp could ignore joint consensus for some time

TimelineCreateRequest should get optional safekeepers field with safekeepers chosen by cplane.

rough order:
- add sk infra, but not enforce confs
- change proto
- add wp proto, but not enforce confs
- implement storconn. It will be used and tested by neon_local.
- implement cplane/storcon integration. Route branch creation/deletion 
  through storcon. Then we can test migration of these branches, hm.
  In principle sk choice from cplane can be removed at this point. 
  However, that would be bad because before import 1) 
  storconn doesn't know about existing project so can't colocate tenants 
  2) neither it knows about capacity. So we could instead allow to set sk 
  set in the branch creation request.
  These cplane -> storconn calls should be under feature flag; 
  rollback is safe.
- finally import existing branches. Then we can drop cplane 
  sk selection code.
  also only at this point wp will always use generations and
  so we can drop 'tli creation on connect'.

## Integration with evicted timelines

## Possible optimizations

`AcceptorRefusal` separate message

Preserving connections (not neede)

multiple joint consensus (not neede)

## Misc

We should use Compute <-> safekeeper protocol change to include other (long
yearned) modifications:
