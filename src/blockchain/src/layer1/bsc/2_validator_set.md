# BSC ValidatorSet

## Basic

```
每块          deposit:本块 gas fee 记给出块者(coinbase)
每 16 块      turn 轮转:换下一个出块验证者(turnLength=16)
每 200 块     distributeFinalityReward:发快速最终性投票奖励
每 1000 块    epoch checkpoint:出块集(21)写进块头 extraData + 重置投票
每 1024 块    Snapshot 对象存盘(checkpointInterval,与 epoch 无关)
每天(呼吸块) updateValidatorSetV2:从 StakeHub 重新选举 45,写入 ValidatorSet 合约
```

## Validators Select

```
Tier A  全部注册/候选集(StakeHub 0x…2002)          — 无上限
   │  getValidatorElectionInfo:返回全部质押验证者 + 投票权 + BLS投票地址
   │  投票权 = StakeCredit.totalPooledBNB(jailed 则记 0)
   │  【呼吸块,每天】按投票权取 top-45
   ▼
Tier B  选举集 45(maxElectedValidators)             — StakeHub.sol:272
   │  = 21 cabinet 池 + 24 candidate 池(按投票权排名切分)
   │  【epoch checkpoint,每 1000 块】getMiningValidators 洗牌
   ▼
Tier C  出块集 21(numOfCabinets)                    — BSCValidatorSet.sol:24
   │  = 18 固定 cabinet + 3 轮换 candidate(3 个工作槽随机化)
   │  【snapshot 激活,checkpoint + 175 块】snap.Validators = 新集合
   ▼
        turn 轮转出块(turnLength=16)
```

1. StakeHub 提供全部候选集合
    * 返回**所有**注册验证者(EnumerableSet 插入序,未排序)
    * `votingPowers[i] = jailed ? 0 : totalPooledBNB()`(自委托 + 全部委托;jailed 记 0 → 自动落选)
2. 客户端取 top-45
    * 过滤 votingPower > 0 → 最大堆 → pop 前 45 个(投票权降序)
3. 写回合约(21/24 是隐式排名切分)
    * 把 45 个(投票权降序)经系统交易发给 `updateValidatorSetV2`(**ValidatorSet 0x…1000**,不是 StakeHub)。
    * 合约 `doUpdateState` **按位置逐个复制,不重排**，`currentValidatorSet` 保持**投票权降序**
    * **21/24 不是存储字段,是排名切分**:前 21 名(质押最高)= cabinet 池,第 22~45 名 = candidate 池
    * **交互方向:READ StakeHub、WRITE ValidatorSet**。排序和 top-N 由客户端做,合约只收成品名单
4. Shuffle:每 epoch 怎么选出 18+3
    * 把位置 18/19/20 各与 `[0,21)` 里随机位置交换 → **任何 cabinet 成员(含质押最高的位置 0)都可能被拉到工作槽 18/19/20**
    * 把位置 18/19/20 各与 `[18,45)` 交换 → 从 candidate 池拉人进工作槽;若被拉进工作槽的 cabinet 成员被换到 ≥21,则**被排除出本 epoch 出块集**
    * **质押最高者入选概率很高但不是 100%**,单 epoch 有小概率被轮空。设计意图是公平——让每个 cabinet 成员都偶尔进入工作槽、分摊与候选竞争的小风险。

## Effective

区分**合约层写入**和**共识层生效**。当前主网 `minerHistoryCheckLen = (21/2+1)×16−1 = 175`。

1. 呼吸块: 更新合约选举集(45),当块生效(仅合约层)
2. epoch checkpoint: 出块集(21)写块头,`number%1000==0` 生效(仅块头层)
3. snapshot 激活: 共识层切换`number%1000==175` 生效(开始出块).

### minerHistoryCheckLen
```
minerHistoryCheckLen = (len(Validators)/2 + 1) × TurnLength − 1
                     = (21/2 + 1) × 16 − 1 = 11 × 16 − 1 = 175
```

半数以上在 FF 失效的情况下，不会回滚到 N/2+1 之前。

## Slash

* **第一层:每次漏块本身的即时损失(连续,不看 count)**
轮到你没出 → 那个块的奖励(`incoming`)记给顶替的备份验证者。**每漏一块都发生**,包括 51-99、101-149。
* **第二层:SlashIndicator 里程碑动作(离散,只在 50/150 整数倍)**

| count | 动作 |
|---|---|
| < 50 | 无里程碑动作(但第一层持续在丢奖励) |
| = 50、100 | **misdemeanor**:把已累积的 `incoming` 清零、**平分给其他所有在役验证者**(`BSCValidatorSet.sol:886-916`)。不罚质押、不封禁 |
| = 150 | **felony**:踢出出块集 + 罚 10 BNB + 封 2 天,count 归零 |

> 不是"51-99 无惩罚"。第一层(丢奖励)一直在发生;misdemeanor/felony 是每 50/150 次结算一次的升级动作。
> `clean()` 每呼吸块把 count 衰减 `150/4≈37`(`SlashIndicator.sol:148`)→ 要触发 felony 得**一天内持续掉线**累积,非终身累计。

| | 罚金 | 封禁 | 举报奖励 |
|---|---|---|---|
| downtime(漏块) | 10 BNB → SystemReward | 2 天 | 无 |
| **double-sign** | **200 BNB** → SystemReward | **30 天** | 举报人得 SystemReward 余额 **20%** |
| **恶意投票** | **200 BNB** → SystemReward | **30 天** | 同上 20% |
