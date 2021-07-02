---
layout: "post"
title: "raft in etcd"
date: "2020-09-10 14:15"
---
## 初始化流程
![](/public/images/boostrap.svg)
```golang
func NewServer(cfg ServerConfig) (srv *EtcdServer, err error) {
  ...
  // 判断是否存在WAL文件，即判断是否为空节点
  haveWAL := wal.Exist(cfg.WALDir())
  ...
  switch {
  // 如果当前节点是新节点，且加入了已存在的cluster
  case !haveWAL && !cfg.NewCluster:
    // 根据配置文件中的PeerURLs创建RaftCluster的内存结构:cl，
    // 为每个Member，生成MemberID（更具PeerURL生成，唯一性）
    // 基于MemberID生成ClusterID
    cl = membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
    ...
    // 按照字母序依次访问PeerURLS，请求“/members”接口，获取当前集群的成员配置:existingCluster。
    // 如果获取失败，则初始化失败。
    existingCluster = GetClusterFromRemotePeers(cfg.Logger, getRemotePeerURLs(cl, cfg.Name), prt)
    // 比较本地生成的Cl与获取的existingcl，如果len(cl) == len(existingCluster)，
    // 同时所有Members PeerURLS相等。则重置本地MemberID为远端ID。否则初始化失败。
    membership.ValidateClusterAndAssignIDs(cfg.Logger, cl, existingCluster)

    remotes = existingCluster.Members()
    // 设置本地clusterID为0，设置cid为remote clusterID
    cl.SetID(types.ID(0), existingCluster.ID())
    // 设置成员配置信息的持久化存储引擎
    cl.SetStore(st)
    cl.SetBackend(be)

    id, n, s, w = startNode(cfg, cl, nil)
    cl.SetID(id, existingCluster.ID())

  // 如果当前节点为新节点，且加入新集群
  case !haveWAL && cfg.NewCluster:
    cl, err = membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
    if err != nil {
      return nil, err
    }
    // 检查当前节点是否已经在集群中启动了，防止重复启动。
    // 启动的标志是当前节点存在于remote membership中，且已经通过publish发布过client urls。
    m := cl.MemberByName(cfg.Name)
    if isMemberBootstrapped(cfg.Logger, cl, cfg.Name, prt, cfg.bootstrapTimeout()) {
      return nil, fmt.Errorf("member %s has already been bootstrapped", m.ID)
    }
    ...
    cl.SetStore(st)
    cl.SetBackend(be)
    // 以配置文件中标定的configuration启动raft节点
    id, n, s, w = startNode(cfg, cl, cl.MemberIDs())
    cl.SetID(id, cl.ID())

  // 当前节点非新节点
  case haveWAL:
    // 以最新的快照启动当前节点,快照需要比committed index更小。（snap_index <= applied <= commiited）
    walSnaps, err := wal.ValidSnapshotEntries(cfg.Logger, cfg.WALDir())
    if err != nil {
      return nil, err
    }
    // 找到对应的snap文件
    snapshot, err := ss.LoadNewestAvailable(walSnaps)
    if err != nil && err != snap.ErrNoSnapshot {
      return nil, err
    }
    if snapshot != nil {
      // snapshor data中存放的是V2Store的内容
      if err = st.Recovery(snapshot.Data); err != nil {
        cfg.Logger.Panic("failed to recover from snapshot", zap.Error(err))
      }
      // 如果本地db对应的applied index比较低，说明db比较就，需要从snap.db中重新加载。
      if be, err = recoverSnapshotBackend(cfg, be, *snapshot, beExist); err != nil {
        cfg.Logger.Panic("failed to recover v3 backend from snapshot", zap.Error(err))
      }
      s1, s2 := be.Size(), be.SizeInUse()
    }
    // 以snapshot中的confstate初始化集群成员配置。
    if !cfg.ForceNewCluster {
      id, cl, n, s, w = restartNode(cfg, snapshot)
    } else {
      // 强制重置集群信息，向log中添加ConfigRemoveNode entries移除除本节点以外
      // 的所有其余节点。添加自身，移除其余。持久化log entries到本地并强制commit
      id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
    }
    cl.SetStore(st)
    cl.SetBackend(be)
    cl.Recover(api.UpdateCapability)

...
tr := &rafthttp.Transport{
  URLs:        cfg.PeerURLs,
  ClusterID:   cl.ID(),
  Raft:        srv,
}
if err = tr.Start(); err != nil {
  return nil, err
}
// add all remotes into transport
for _, m := range remotes {
  if m.ID != id {
    tr.AddRemote(m.ID, m.PeerURLs)
  }
}
for _, m := range cl.Members() {
  if m.ID != id {
    tr.AddPeer(m.ID, m.PeerURLs)
  }
}
srv.r.transport = tr

return srv, nil
}
```

WAL文件

文件名：seq-index.wal
文件内容：
MetaData：
Entry：[term,index,entryType,data] entryType包括 normal,confchange。
State: 记录Raft的持久化状态信息（current term, voted for以及committed index）。
Crc: 记录文件起始到目前为止的crc校验和
Snapshot: 记录index和term。

Snap文件

文件名：term-index.snap
文件内容：当前term，当前index，当前confState，额外Data

Snap.DB文件

文件名：index.snap.db
文件内容：DB的快照

```golang
func startNode(cfg ServerConfig, cl *membership.RaftCluster, ids []types.ID) (id types.ID, n raft.Node, s *raft.MemoryStorage, w *wal.WAL) {
  member := cl.MemberByName(cfg.Name)
  metadata := pbutil.MustMarshal(
    &pb.Metadata{
      NodeID:    uint64(member.ID),
      ClusterID: uint64(cl.ID()),
    },
  )
  // 以自身MemberID和remote clusterID作为Meta信息，创建WAL文件
  if w, err = wal.Create(cfg.Logger, cfg.WALDir(), metadata); err != nil {
    cfg.Logger.Panic("failed to create WAL", zap.Error(err))
  }
  // 根据传入的MemberID，构建Peer
  peers := make([]raft.Peer, len(ids))
  for i, id := range ids {
    var ctx []byte
    ctx, err = json.Marshal((*cl).Member(id))
    if err != nil {
      cfg.Logger.Panic("failed to marshal member", zap.Error(err))
    }
    peers[i] = raft.Peer{ID: uint64(id), Context: ctx}
  }
  s = raft.NewMemoryStorage()
  c := &raft.Config{
    ID:              uint64(id),
    ElectionTick:    cfg.ElectionTicks,
    HeartbeatTick:   1,
    Storage:         s,
    MaxSizePerMsg:   maxSizePerMsg,
    MaxInflightMsgs: maxInflightMsgs,
    CheckQuorum:     true,
    PreVote:         cfg.PreVote,
  }
  // 启动Raft一致性模块
  if len(peers) == 0 {
    n = raft.RestartNode(c)
  } else {
    n = raft.StartNode(c, peers)
  }
  return id, n, s, w
}
```

```golang
// 启动节点，成员配置为空
func RestartNode(c *Config) Node {
  rn, err := NewRawNode(c)
  if err != nil {
    panic(err)
  }
  n := newNode(rn)
  go n.run()
  return &n
}

// 启动节点，成员配置不为空
func StartNode(c *Config, peers []Peer) Node {
  if len(peers) == 0 {
    panic("no peers given; use RestartNode instead")
  }
  rn, err := NewRawNode(c)
  if err != nil {
    panic(err)
  }
  rn.Bootstrap(peers)

  n := newNode(rn)

  go n.run()
  return &n
}

// config storage存储confstate进行成员状态恢复
func NewRawNode(config *Config) (*RawNode, error) {
  r := newRaft(config)
  rn := &RawNode{
    raft: r,
  }
  rn.prevSoftSt = r.softState()
  rn.prevHardSt = r.hardState()
  return rn, nil
}

func (rn *RawNode) Bootstrap(peers []Peer) error {
  // 根据提供的peer信息，生成configuration entries
  rn.raft.becomeFollower(1, None)
  ents := make([]pb.Entry, len(peers))
  for i, peer := range peers {
    cc := pb.ConfChange{Type: pb.ConfChangeAddNode, NodeID: peer.ID, Context: peer.Context}
    data, err := cc.Marshal()
    if err != nil {
      return err
    }

    ents[i] = pb.Entry{Type: pb.EntryConfChange, Term: 1, Index: uint64(i + 1), Data: data}
  }
  rn.raft.raftLog.append(ents...)

  // 应用日志，当前成员配置日志并未进行持久化，且未通过大多数确认。但仍然会同步给followers。
  // 违背了leader完整性特性，新主可能丢失。
  rn.raft.raftLog.committed = uint64(len(ents))
  for _, peer := range peers {
    rn.raft.applyConfChange(pb.ConfChange{NodeID: peer.ID, Type: pb.ConfChangeAddNode}.AsV2())
  }
  return nil
}

func newRaft(c *Config) *raft {
  // 从storage中恢复信息（committed index， current term， voted for和成员配置信息）
  hs, cs, err := c.Storage.InitialState()

  r := &raft{
  id:                        c.ID,
  lead:                      None,
  isLearner:                 false,
  raftLog:                   raftlog,
  maxMsgSize:                c.MaxSizePerMsg,
  maxUncommittedSize:        c.MaxUncommittedEntriesSize,
  prs:                       tracker.MakeProgressTracker(c.MaxInflightMsgs),
  electionTimeout:           c.ElectionTick,
  heartbeatTimeout:          c.HeartbeatTick,
  logger:                    c.Logger,
  checkQuorum:               c.CheckQuorum,
  preVote:                   c.PreVote,
  readOnly:                  newReadOnly(c.ReadOnlyOption),
  disableProposalForwarding: c.DisableProposalForwarding,
  }

  // 根据当前成员配置状态，生成cfg和prs（还未应用到raft.ProgressTracker中）
  cfg, prs, err := confchange.Restore(confchange.Changer{
  Tracker:   r.prs,
  LastIndex: raftlog.lastIndex(),
  }, cs)
  if err != nil {
    panic(err)
  }

  assertConfStatesEquivalent(r.logger, cs, r.switchToConfig(cfg, prs))

  if !IsEmptyHardState(hs) {
    r.loadState(hs)
  }
  if c.Applied > 0 {
    raftlog.appliedTo(c.Applied)
  }
  r.becomeFollower(r.Term, None)

  return r
}

// 返回变更过后的成员配置信息。
func (r *raft) switchToConfig(cfg tracker.Config, prs tracker.ProgressMap) pb.ConfState {
  // 根据提供的config和progressMap重置当前配置。
  r.prs.Config = cfg
  r.prs.Progress = prs

  cs := r.prs.ConfState()
  pr, ok := r.prs.Progress[r.id]

  // Update whether the node itself is a learner, resetting to false when the
  // node is removed.
  r.isLearner = ok && pr.IsLearner

  // 如果在新成员配置中，当前被移除或者降级，直接返回
  if (!ok || r.isLearner) && r.state == StateLeader {
    return cs
  }

  // The remaining steps only make sense if this node is the leader and there
  // are other nodes.
  if r.state != StateLeader || len(cs.Voters) == 0 {
    return cs
  }
  if r.maybeCommit() {
    // If the configuration change means that more entries are committed now,
    // broadcast/append to everyone in the updated config.
    r.bcastAppend()
  } else {
    // Otherwise, still probe the newly added replicas; there's no reason to
    // let them wait out a heartbeat interval (or the next incoming
    // proposal).
    r.prs.Visit(func(id uint64, pr *tracker.Progress) {
      r.maybeSendAppend(id, false /* sendIfEmpty */)
    })
  }
  // 如果当前节点正在进行leader transfer，但是被交接者在新配置中，已经被移除了，则停止交接。
  if _, tOK := r.prs.Config.Voters.IDs()[r.leadTransferee]; !tOK && r.leadTransferee != 0 {
    r.abortLeaderTransfer()
  }
  return cs
}
```

```golang
func Restore(chg Changer, cs pb.ConfState) (tracker.Config, tracker.ProgressMap, error) {
  outgoing, incoming := toConfChangeSingle(cs)

  var ops []func(Changer) (tracker.Config, tracker.ProgressMap, error)

  if len(outgoing) == 0 {
    // 成员配置并非处于joint consensus状态，可以采用simple change方式处理
    for _, cc := range incoming {
      cc := cc
      ops = append(ops, func(chg Changer) (tracker.Config, tracker.ProgressMap, error) {
        return chg.Simple(cc)
      })
    }
  } else {
    // The ConfState describes a joint configuration.
    //
    // First, apply all of the changes of the outgoing config one by one, so
    // that it temporarily becomes the incoming active config. For example,
    // if the config is (1 2 3)&(2 3 4), this will establish (2 3 4)&().
    for _, cc := range outgoing {
      cc := cc // loop-local copy
      ops = append(ops, func(chg Changer) (tracker.Config, tracker.ProgressMap, error) {
        return chg.Simple(cc)
      })
    }
    // Now enter the joint state, which rotates the above additions into the
    // outgoing config, and adds the incoming config in. Continuing the
    // example above, we'd get (1 2 3)&(2 3 4), i.e. the incoming operations
    // would be removing 2,3,4 and then adding in 1,2,3 while transitioning
    // into a joint state.
    ops = append(ops, func(chg Changer) (tracker.Config, tracker.ProgressMap, error) {
      return chg.EnterJoint(cs.AutoLeave, incoming...)
    })
  }

  return chain(chg, ops...)
}

func chain(chg Changer, ops ...func(Changer) (tracker.Config, tracker.ProgressMap, error)) (tracker.Config, tracker.ProgressMap, error) {
  for _, op := range ops {
    cfg, prs, err := op(chg)
    if err != nil {
      return tracker.Config{}, nil, err
    }
    chg.Tracker.Config = cfg
    chg.Tracker.Progress = prs
  }
  return chg.Tracker.Config, chg.Tracker.Progress, nil
}

// out代表针对outgoing配置需要做的变更，in代表针对incoming配置需要做的变更
func toConfChangeSingle(cs pb.ConfState) (out []pb.ConfChangeSingle, in []pb.ConfChangeSingle) {
  // 如果VotersOutgoing不为空，说明我们将采用joint consensus方式进行成员配置变更。
  for _, id := range cs.VotersOutgoing {
    // If there are outgoing voters, first add them one by one so that the
    // (non-joint) config has them all.
    out = append(out, pb.ConfChangeSingle{
      Type:   pb.ConfChangeAddNode,
      NodeID: id,
    })
  }
  for _, id := range cs.VotersOutgoing {
    in = append(in, pb.ConfChangeSingle{
      Type:   pb.ConfChangeRemoveNode,
      NodeID: id,
    })
  }
  // Then we'll add the incoming voters and learners.
  for _, id := range cs.Voters {
    in = append(in, pb.ConfChangeSingle{
      Type:   pb.ConfChangeAddNode,
      NodeID: id,
    })
  }
  for _, id := range cs.Learners {
    in = append(in, pb.ConfChangeSingle{
      Type:   pb.ConfChangeAddLearnerNode,
      NodeID: id,
    })
  }
  // Same for LearnersNext; these are nodes we want to be learners but which
  // are currently voters in the outgoing config.
  for _, id := range cs.LearnersNext {
    in = append(in, pb.ConfChangeSingle{
      Type:   pb.ConfChangeAddLearnerNode,
      NodeID: id,
    })
  }
  return out, in
}
```


```golang
/*config changer related*/

// 持久化的信息。
type ConfState struct {
  // new voters
  Voters []uint64
  // new learners
  Learners []uint64

  // 以下fields在joint config时起作用
  // old voters
  VotersOutgoing []uint64
  // 表明离开joint之后，哪些old voters降级为learners，其余old voters
  // 则不再集群中扮演任何角色。
  LearnersNext []uint64
  AutoLeave        bool
}
// 例子：voters=(1 2 3) learners=(5) outgoing=(1 2 4 6) learners_next=(4)
// 在进入jonit consensus之前，当前局集群的voters为 （1 2 4 6），learners 为 （5）.
// 新的voters为（1 2 3），即（1 2）节点将被保留，同时（4 6）节点将不再成为voters，
// 特别地，节点（4）将降级为learner。节点（3， 5）将新添加进集群


type Config struct {
  // outgoing代表old configuration， incoming代表new configuration。
  // [2]MajorityConfig, [0]代表 incoming，[1]代表outgoing
  // 对于simple change方式来说，voters变更始终操作voters[0]，即incoming config
  // 对于joint consensus方式来说，voters[1]才有意义，代表old configuration。
  Voters quorum.JointConfig
  ...
}

// 跟踪当前生效的成员配置，更新其同步状态。
type ProgressTracker struct {
  Config

  Progress ProgressMap

  Votes map[uint64]bool

  MaxInflight int
}

// 管理成员变更事件
type Changer struct {
  Tracker   tracker.ProgressTracker
  LastIndex uint64
}

// 针对单成员的一次变更行为（新增/移除一个成员，提升一个成员身份）
type ConfChangeSingle struct {
  Type             ConfChangeType `protobuf:"varint,1,opt,name=type,enum=raftpb.ConfChangeType" json:"type"`
  NodeID           uint64         `protobuf:"varint,2,opt,name=node_id,json=nodeId" json:"node_id"`
  XXX_unrecognized []byte         `json:"-"`
}

// 根据当前成员配置信息，生成一份Joint entry，包含（old configuration和new configuration）
func (c Changer) EnterJoint(autoLeave bool, ccs ...pb.ConfChangeSingle) (tracker.Config, tracker.ProgressMap, error) {
  // 拷贝当前的成员配置状态，包括voters，learners，learnersNext。
  cfg, prs, err := c.checkAndCopy()
  if err != nil {
    return c.err(err)
  }
  // 如果outgoing不为空，说明已经进入joint consensus状态，退出
  if joint(cfg) {
    err := errors.New("config is already joint")
    return c.err(err)
  }
  // 如果incoming为空，则说明最终成员配置将没有任何投票者，即无法选主
  if len(incoming(cfg.Voters)) == 0 {
    err := errors.New("can't make a zero-voter config joint")
    return c.err(err)
  }
  // 清空outgoing，复制incoming到outgoing中。
  // 因为一旦采用joint consensus方式变更成员，当前的成员配置（incoming）都将变为old configuration
  *outgoingPtr(&cfg.Voters) = quorum.MajorityConfig{}
  for id := range incoming(cfg.Voters) {
    outgoing(cfg.Voters)[id] = struct{}{}
  }

  // 根据输入ccs，调整incoming为最终的成员配置形态。
  if err := c.apply(&cfg, prs, ccs...); err != nil {
    return c.err(err)
  }
  cfg.AutoLeave = autoLeave
  return checkAndReturn(cfg, prs)
}

func (c Changer) LeaveJoint() (tracker.Config, tracker.ProgressMap, error) {
  cfg, prs, err := c.checkAndCopy()
  if err != nil {
    return c.err(err)
  }
  if !joint(cfg) {
    err := errors.New("can't leave a non-joint config")
    return c.err(err)
  }
  if len(outgoing(cfg.Voters)) == 0 {
    err := fmt.Errorf("configuration is not joint: %v", cfg)
    return c.err(err)
  }
  // 在old configuration中以voter身份存在的节点，现在可以
  // 切换为learners了。
  for id := range cfg.LearnersNext {
    nilAwareAdd(&cfg.Learners, id)
    prs[id].IsLearner = true
  }
  cfg.LearnersNext = nil

  // 清理不在new configuration中的节点，这些节点不需要再同步日志了，
  // 可以从集群中安全的移除了。
  for id := range outgoing(cfg.Voters) {
    _, isVoter := incoming(cfg.Voters)[id]
    _, isLearner := cfg.Learners[id]

    if !isVoter && !isLearner {
      delete(prs, id)
    }
  }
  *outgoingPtr(&cfg.Voters) = nil
  cfg.AutoLeave = false

  return checkAndReturn(cfg, prs)
}

// simple change方式，一次只允许改变一个voters，否则应该使用joint consensus方式。
func (c Changer) Simple(ccs ...pb.ConfChangeSingle) (tracker.Config, tracker.ProgressMap, error) {
  cfg, prs, err := c.checkAndCopy()
  if err != nil {
    return c.err(err)
  }
  if joint(cfg) {
    err := errors.New("can't apply simple config change in joint config")
    return c.err(err)
  }
  if err := c.apply(&cfg, prs, ccs...); err != nil {
    return c.err(err)
  }
  if n := symdiff(incoming(c.Tracker.Voters), incoming(cfg.Voters)); n > 1 {
    return tracker.Config{}, nil, errors.New("more than one voter changed without entering joint config")
  }

  return checkAndReturn(cfg, prs)
}

// 应用成员变更信息到incoming config中。
// 通过apply，最终得到一个prs和cfg匹配的同步配置。
// 任何一个成员（无论角色，voters，learners，learnersNext）都需要对应一个Progress
func (c Changer) apply(cfg *tracker.Config, prs tracker.ProgressMap, ccs ...pb.ConfChangeSingle) error {
  for _, cc := range ccs {
    if cc.NodeID == 0 {
      // etcd replaces the NodeID with zero if it decides (downstream of
      // raft) to not apply a change, so we have to have explicit code
      // here to ignore these.
      continue
    }
    switch cc.Type {
    case pb.ConfChangeAddNode:
      // 添加进incoimg voters或者cfg.learners中。
      c.makeVoter(cfg, prs, cc.NodeID)
    case pb.ConfChangeAddLearnerNode:
      // 1）如果目前节点不存在。表明新加入节点，则加入cfg.learners中
      //
      // 2)如果节点以voters存在与当前配置中，说明该节点在最终配置中需要降级为learner，
      // remove该节点，保留log Progress, 添加进LearnersNext。
      //
      // 3) 如果节点以learners存在于当前配置中，直接返回。
      c.makeLearner(cfg, prs, cc.NodeID)
    case pb.ConfChangeRemoveNode:
      // 移除该节点的配置信息（从incoming voters，learners, learners_next中移除）
      //
      // 1) simple change模式下，直接移除其Progress（不再同步复制，不再彩玉投票，如果当前为voter）
      // 2) 在joint consensus方式下，如果该节点在old configuration中以voter存在，为了保证正常服务
      // （需要参与投票）仍然需要保留其log Progress。
      c.remove(cfg, prs, cc.NodeID)
    case pb.ConfChangeUpdateNode:
    default:
      return fmt.Errorf("unexpected conf type %d", cc.Type)
    }
  }
  // 如果新配置中，不存在任何投票者，说明不能提供服务，当前配置不允许产生。
  if len(incoming(cfg.Voters)) == 0 {
    return errors.New("removed all voters")
  }
  return nil
}

func (c Changer) checkAndCopy() (tracker.Config, tracker.ProgressMap, error) {
  cfg := c.Tracker.Config.Clone()
  prs := tracker.ProgressMap{}

  for id, pr := range c.Tracker.Progress {
    // A shallow copy is enough because we only mutate the Learner field.
    ppr := *pr
    prs[id] = &ppr
  }
  return checkAndReturn(cfg, prs)
}

func checkAndReturn(cfg tracker.Config, prs tracker.ProgressMap) (tracker.Config, tracker.ProgressMap, error) {
  // 保证Progress和Config的一致性，每个Config中的成员都存在一个Progress记录
  if err := checkInvariants(cfg, prs); err != nil {
    return tracker.Config{}, tracker.ProgressMap{}, err
  }
  return cfg, prs, nil
}
```

## Snapshot
### Follower接收快照

```golang
func (r *raft) handleSnapshot(m pb.Message) {
  sindex, sterm := m.Snapshot.Metadata.Index, m.Snapshot.Metadata.Term
  if r.restore(m.Snapshot) {
    r.logger.Infof("%x [commit: %d] restored snapshot [index: %d, term: %d]",
      r.id, r.raftLog.committed, sindex, sterm)
    r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.lastIndex()})
  } else {
    r.logger.Infof("%x [commit: %d] ignored snapshot [index: %d, term: %d]",
      r.id, r.raftLog.committed, sindex, sterm)
    r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
  }
}

func (r *raft) restore(s pb.Snapshot) bool {
  // 如果本地已提交日志Index大于快照的Index，说明这是过期的快照，快速返回。
  if s.Metadata.Index <= r.raftLog.committed {
    return false
  }
  ...
  // 如果快照Index和Term在本地匹配，则快速提交本地日志并返回。
  // 说明本地已包含快照数据对应的日志，只是还未提交。
  if r.raftLog.matchTerm(s.Metadata.Index, s.Metadata.Term) {
    r.raftLog.commitTo(s.Metadata.Index)
    return false
  }

  // 本地日志和快照数据不匹配（缺少部分日志或者存在冲突），则以快照数据为准
  // 重置日志信息。
  r.raftLog.restore(s)

  // 根据快照中的ConfState，重置ProgressTrackers
  ...
}

func (l *raftLog) restore(s pb.Snapshot) {
  // 设置committedIndex为快照Index
  l.committed = s.Metadata.Index
  l.unstable.restore(s)
}

func (u *unstable) restore(s pb.Snapshot) {
  u.offset = s.Metadata.Index + 1
  u.entries = nil
  u.snapshot = &s
}
```
**Follower**接收到Leader传递过来的快照数据，首先根据当前的本地日志状态，判断是否需要应用
该快照数据。若需要应用，则首先存储在unstable log中，等待上层模块下次调用Ready接口返回快照。

```golang
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
  ...
  if r.raftLog.unstable.snapshot != nil {
    rd.Snapshot = *r.raftLog.unstable.snapshot
  }
  ...
  return rd
}
```
上层应用从Raft协议模块拿到了unstable数据，则进行持久化处理，并更新相关的内存视图。
```golang
func (r *raftNode) start(rh *raftReadyHandler) {
  select {
      case <-r.ticker.C:
        r.tick()
      case rd := <-r.Ready():

        // 将committed但是未apply的数据打包，给上层（etcdserver）异步线程处理
        // notifyc用于通知上层，数据已经持久化完成了，可以安全的进行相关操作（打快照）。
        notifyc := make(chan struct{}, 1)
        ap := apply{
          entries:  rd.CommittedEntries,
          snapshot: rd.Snapshot,
          notifyc:  notifyc,
        }
        select {
          case r.applyc <- ap:
          case <-r.stopped:
            return
        }


        ...
        // 首先，持久化快照。
        // 1) 首先创建snap文件（文件格式：term-index.snap），并写入snap data
        // 2) 完成snap文件写入之后。在WAL中，写入snapshot entry。用于下次启动自检
        r.storage.SaveSnap(rd.Snapshot)

        // 将HardState（commttedIndex， voterfor， term）写入WAL文件
        // 将未持久化的Entries（Leader收到的客户端请求，Follower收到的MsgAppend），写入WAL文件
        r.storage.Save(rd.HardState, rd.Entries)

        // 到此为止，snapshot, Entries都已经持久化到WAl文件了，则更新内存中的stable log视图
        // 上层引用（etcdserver），通过raftStorage查看当前状态。
        r.raftStorage.ApplySnapshot(rd.Snapshot)
        r.raftStorage.Append(rd.Entries)
        ...
  }
}

func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply) {
  // 如果收到来日Leader的快照数据，首先恢复快照数据，
  s.applySnapshot(ep, apply)
  // 应用已提交的日志
  s.applyEntries(ep, apply)

  // 当前等待回复的客户端，可以进行操作了。
  s.applyWait.Trigger(ep.appliedi)

  <-apply.notifyc

  // 如果 appliedi-snapi > s.Cfg.SnapshotCount 则创建快照
  s.triggerSnapshot(ep)
  select {
  case m := <-s.r.msgSnapC:
    merged := s.createMergedSnapshotMessage(m, ep.appliedt, ep.appliedi, ep.confState)
    s.sendMergedSnap(merged)
  default:
  }
}

func (s *EtcdServer) applySnapshot(ep *etcdProgress, apply *apply) {
  if raft.IsEmptySnap(apply.snapshot) {
    return
  }
  // wait for raftNode to persist snapshot onto the disk
  <-apply.notifyc

  // 更新快照内容，重新加载DB文件（文件名：SnapshotIndex.snap.db)，恢复应用状态机数据
  // 重新建立节点之间的链接，更新appliedIndex等信息。
  openSnapshotBackend(s.Cfg, s.snapshotter, apply.snapshot)
  ...
}
```

### 创建本地快照

每当本轮数据应用完成之后，若appliedi-snapi足够大，即触发创建快照操作。

```golang
func (s *EtcdServer) triggerSnapshot(ep *etcdProgress) {
  if ep.appliedi-ep.snapi <= s.Cfg.SnapshotCount {
    return
  }
  s.snapshot(ep.appliedi, ep.confState)
  ep.snapi = ep.appliedi
}

func (s *EtcdServer) snapshot(snapi uint64, confState raftpb.ConfState) {
  // 提交当前可能未未持久化的DB数据（consistent index）
  s.KV().Commit()

  // 异步创建快照数据
  // 不需要发送给对端，对端需要的快照信息（Term，Index，ConfState等）已经准备好。
  // 创建快照的目的是为了加速本节点重启后的状态恢复，以及裁剪日志文件。
  go func(){
    // 创建快照
    s.r.raftStorage.CreateSnapshot(snapi, &confState, d)
    // 持久化快照数据
    s.r.storage.SaveSnap(snap)
    // compact compacti之前的所有数据，释放内存
    // WAL文件的裁剪，由后台线程处理
    s.r.raftStorage.Compact(compacti)
  }()
```
