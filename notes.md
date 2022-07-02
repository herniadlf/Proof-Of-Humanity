# Some notes about how PoH uses an arbitrator

## What we need if we want to change arbitror?

In order to change arbitrator for Proof Of Humanity, we need:

### A new arbitror contract
Simil KlerosLiquid but using UBI or UBIVOTE token address. The constructor has:

- address _governor --> A Governor Contract. Currently is [KlerosGovernor](https://etherscan.io/address/0xe5bcea6f87aaee4a81f64dfdb4d30d400e0e5cf4#code). We will need the POH Governor contract
- Pinakion _pinakion --> Not Pinakion, UBI or UBIVOTE
- RNG _RNGenerator --> A contract that can give us Random Numbers. Currently: [BlockHashRNGFallback](https://etherscan.io/address/0x1738b62e403090666687243e758b1c29edffc90e)
- uint _minStakingTime, --> currently 3600 uint256
- uint _maxDrawingTime, --> currently 7200 uint256
- bool _hiddenVotes, --> I suppose "true".
- uint _minStake, --> TO BE RESEARCHED
- uint _alpha, --> TO BE RESEARCHED
- uint _feeForJuror, --> TO BE RESEARCHED
- uint _jurorsForCourtJump, --> TO BE RESEARCHED
- uint[4] _timesPerPeriod, --> TO BE RESEARCHED
- uint _sortitionSumTreeK  --> TO BE RESEARCHED

### A new governor (?)

Nowadays, the PoH Governor and the KlerosLiquid Governor is the same contract: KlerosGovernor.

## Pinakion uses on KlerosLiquid

Okay, now let's follow the pinakion token. It's declared:

```solidity
import { MiniMeTokenERC20 as Pinakion } from "@kleros/kleros-interaction/contracts/standard/arbitration/ArbitrableTokens/MiniMeTokenERC20.sol";
```

Maybe we should understand the MiniMeTokenERC20. I think that it's something about cloning and updating the token.

The pinakion is a contract atribute. The address is configured on the constructor, but it also can be updated with changePinakion method.

```solidity
Pinakion public pinakion; // The Pinakion token contract.

/** @dev Changes the `pinakion` storage variable.
     *  @param _pinakion The new value for the `pinakion` storage variable.
     */
function changePinakion(Pinakion _pinakion) external onlyByGovernor {
    pinakion = _pinakion;
}
```

Uses in `execute` which split rewards to jurors or collect penalties from jurors:

```solidity
                // Juror was active, and voted coherently or it was a tie.
                if (i >= dispute.votes[_appeal].length) { // Only execute in the second half of the iterations.

                    // Reward.
                    pinakion.transfer(vote.account, tokenReward);
```
```solidity
                // Juror was inactive, or voted incoherently and it was not a tie.
                if (i < dispute.votes[_appeal].length) { // Only execute in the first half of the iterations.

                    // Penalize.
                    uint penalty = dispute.tokensAtStakePerJuror[_appeal] > pinakion.balanceOf(vote.account) ? pinakion.balanceOf(vote.account) : dispute.tokensAtStakePerJuror[_appeal];

                    pinakion.transferFrom(vote.account, this, penalty); <-------- HERE

                    emit TokenAndETHShift(vote.account, _disputeID, -int(penalty), 0);
                    penaltiesInRoundCache += penalty;
                    jurors[vote.account].lockedTokens -= dispute.tokensAtStakePerJuror[_appeal];

                    // Unstake juror if his penalty made balance less than his total stake or if he lost due to inactivity.

                    if (pinakion.balanceOf(vote.account) < jurors[vote.account].stakedTokens || !vote.voted) <---- HERE
                        for (uint j = 0; j < jurors[vote.account].subcourtIDs.length; j++)
                            _setStake(vote.account, jurors[vote.account].subcourtIDs[j], 0);
```
```solidity
                // Send fees and tokens to the governor if no one was coherent.
                if (dispute.votesInEachRound[_appeal] == 0 || !dispute.voteCounters[dispute.voteCounters.length - 1].tied && dispute.voteCounters[_appeal].counts[dispute.voteCounters[dispute.voteCounters.length - 1].winningChoice] == 0) {
                    // Intentional use to avoid blocking.
                    governor.send(dispute.totalFeesForJurors[_appeal]); // solium-disable-line security/no-send
                    
                    pinakion.transfer(governor, penaltiesInRoundCache); <------ HERE
                } else if (i + 1 < end) {
                    // Compute rewards because we are going into rewarding.
                    dispute.penaltiesInEachRound[_appeal] = penaltiesInRoundCache;
                    (tokenReward, ETHReward) = computeTokenAndETHRewards(_disputeID, _appeal);
                }
```

Then we can see it in the setStake function. `currentStake` is readed from the sortition tree, `_stake` is the current amount willing to stake.
```solidity
        uint newTotalStake = juror.stakedTokens - currentStake + _stake; // Can't overflow because _stake is a uint128.
        if (!(_stake == 0 || pinakion.balanceOf(_account) >= newTotalStake))
            return false; // The juror's total amount of staked tokens cannot be higher than the juror's balance.

```

There is a use in a method that I don't understand:

```solidity
/** @dev Notifies the controller about a token transfer allowing the controller to react if desired.
     *  @param _from The origin of the transfer.
     *  @param _to The destination of the transfer.
     *  @param _amount The amount of the transfer.
     *  @return allowed Whether the operation should be allowed or not.
     */
    function onTransfer(address _from, address _to, uint _amount) public returns(bool allowed) {
        if (lockInsolventTransfers) { // Never block penalties or rewards.
            uint newBalance = pinakion.balanceOf(_from) - _amount;
            if (newBalance < jurors[_from].stakedTokens || newBalance < jurors[_from].lockedTokens) return false;
        }
        allowed = true;
    }
```

### Kleros liquid. Disputes, periods, phases, stake, sortition Tree, etc...
Here, the dispute has been created on the Humanity Court with the evidence. Now, we better move to Kleros Liquid to understand how this continue.

Disputes has this periods:

```solidity
enum Period {
      evidence, // Evidence can be submitted. This is also when drawing has to take place.
      commit, // Jurors commit a hashed vote. This is skipped for courts without hidden votes.
      vote, // Jurors reveal/cast their vote depending on whether the court has hidden votes or not.
      appeal, // The dispute can be appealed.
      execution // Tokens are redistributed and the ruling is executed.
    }
```

Disputes are created with the first period: EVIDENCE.

During this period, the evidence moves off-chain. Meanwhile, in the court we need the stakes to draw juror/jurors.

First of all, KlerosLiquid works with 3 phases: STAKING, GENERATING and DRAWING. This is the method that pass them:
```solidity
/** @dev Passes the phase. TRUSTED */
    function passPhase() external {
        if (phase == Phase.staking) {
            require(now - lastPhaseChange >= minStakingTime, "The minimum staking time has not passed yet.");
            require(disputesWithoutJurors > 0, "There are no disputes that need jurors.");
            RNBlock = block.number + 1;
            RNGenerator.requestRN(RNBlock);
            phase = Phase.generating;
        } else if (phase == Phase.generating) {
            RN = RNGenerator.getUncorrelatedRN(RNBlock);
            require(RN != 0, "Random number is not ready yet.");
            phase = Phase.drawing;
        } else if (phase == Phase.drawing) {
            require(disputesWithoutJurors == 0 || now - lastPhaseChange >= maxDrawingTime, "There are still disputes without jurors and the maximum drawing time has not passed yet.");
            phase = Phase.staking;
        }

        lastPhaseChange = now;
        emit NewPhase(phase);
    }
```

I think that once there are disputes without jurors, a random number is requested. Then, we are in generation phase.

Once provided the random number (RN), we can go to phase drawing. Here we have a time to draw jurors.
## Drawing jurors

### set Stake
Let's see first how a juror stake PNK's and then how the drawJuror method works. Read about [Sortition Trees](https://medium.com/kleros/an-efficient-data-structure-for-blockchain-sortition-15d202af3247)

```solidity
/** @dev Sets the specified juror's stake in a subcourt.
     *  `O(n + p * log_k(j))` where
     *  `n` is the number of subcourts the juror has staked in,
     *  `p` is the depth of the subcourt tree,
     *  `k` is the minimum number of children per node of one of these subcourts' sortition sum tree,
     *  and `j` is the maximum number of jurors that ever staked in one of these subcourts simultaneously.
     *  @param _account The address of the juror.
     *  @param _subcourtID The ID of the subcourt.
     *  @param _stake The new stake.
     *  @return succeeded True if the call succeeded, false otherwise.
     */
    function _setStake(address _account, uint96 _subcourtID, uint128 _stake) internal returns(bool succeeded) {
        if (!(_subcourtID < courts.length))
            return false;

        // Delayed action logic.
        if (phase != Phase.staking) {
            delayedSetStakes[++lastDelayedSetStake] = DelayedSetStake({ account: _account, subcourtID: _subcourtID, stake: _stake });
            return true;
        }
        
        if (!(_stake == 0 || courts[_subcourtID].minStake <= _stake))
            return false; // The juror's stake cannot be lower than the minimum stake for the subcourt.
        Juror storage juror = jurors[_account];
        // A sortition tree is used for every court. It's used to save stake efficiently. 
        bytes32 stakePathID = accountAndSubcourtIDToStakePathID(_account, _subcourtID);
        // If the juror has no stake, currentStake is 0
        uint currentStake = sortitionSumTrees.stakeOf(bytes32(_subcourtID), stakePathID);
        if (!(_stake == 0 || currentStake > 0 || juror.subcourtIDs.length < MAX_STAKE_PATHS))
            return false; // Maximum stake paths reached.
        uint newTotalStake = juror.stakedTokens - currentStake + _stake; // Can't overflow because _stake is a uint128.
        
        // Here appears the beloved PINAKION. It verifies that the juror has enough balance for the staking. 
        if (!(_stake == 0 || pinakion.balanceOf(_account) >= newTotalStake))
            return false; // The juror's total amount of staked tokens cannot be higher than the juror's balance.

        // Update juror's records.
        juror.stakedTokens = newTotalStake;
        if (_stake == 0) {
            for (uint i = 0; i < juror.subcourtIDs.length; i++)
                if (juror.subcourtIDs[i] == _subcourtID) {
                    juror.subcourtIDs[i] = juror.subcourtIDs[juror.subcourtIDs.length - 1];
                    juror.subcourtIDs.length--;
                    break;
                }
        } else if (currentStake == 0) juror.subcourtIDs.push(_subcourtID);

        // Update subcourt parents. I dont know if this is needed for POH 
        bool finished = false;
        uint currentSubcourtID = _subcourtID;
        while (!finished) {
            sortitionSumTrees.set(bytes32(currentSubcourtID), _stake, stakePathID);
            if (currentSubcourtID == 0) finished = true;
            else currentSubcourtID = courts[currentSubcourtID].parent;
        }
        emit StakeSet(_account, _subcourtID, _stake, newTotalStake);
        return true;
    }
```

Now, a new POH Registered, UBI holder is a juror. We need to draw jurors for our friend dispute.

### Draw juror for dispute

```solidity
    function drawJurors(
        uint _disputeID,
        uint _iterations
    ) external onlyDuringPhase(Phase.drawing) onlyDuringPeriod(_disputeID, Period.evidence) {
```

In this method the magic happens. The sortition tree is used to draw a juror. Here, there is a communication with de SortitionTree contract.

See [a transaction example](https://etherscan.io/tx/0x6a7604c20f379a712f6e83994a7e03b2b7c243e838bd21801fd7640091e8cd4d).

The vote is created. There is no a vote, just the "object". The stake is now locked.

```solidity
            // Save the vote.
            dispute.votes[dispute.votes.length - 1][i].account = drawnAddress;
            jurors[drawnAddress].lockedTokens += dispute.tokensAtStakePerJuror[dispute.tokensAtStakePerJuror.length - 1];
```

Boring things happen and the votes are casted and the winning choice is calculated.

The dispute passPeriod is called, imagine that you can execute a ruled dispute. Check what happen once the rules are executed. See [executeRuling](https://etherscan.io/tx/0x86878c73e3a7851a33281924b4a2a257f8b44f0a981da1c574d1e0ed4251bfda)

```solidity
        dispute.ruled = true;
        uint winningChoice = dispute.voteCounters[dispute.voteCounters.length - 1].tied ? 0
            : dispute.voteCounters[dispute.voteCounters.length - 1].winningChoice;
        dispute.arbitrated.rule(_disputeID, winningChoice);
```

Here is something very very interesting. The rule is a arbitrated concern. In this case, POH contract. It makes sense, because the dispute result has impact on POH ecosystem.

### Sending rewards or collecting penalties

Although, the reward shift happen on `execute` method. I think that this is a delayed action. This variables are defined and then used to transfer to juror's wallet:
`(uint tokenReward, uint ETHReward) = (0, 0);`. You can see something like:
```solidity
                    // Reward.
                    pinakion.transfer(vote.account, tokenReward);
                    // Intentional use to avoid blocking.
                    vote.account.send(ETHReward); // solium-disable-line security/no-send
```

Or as a penalty, doing transferFROM:
```solidity
                // Penalize.
                uint penalty = dispute.tokensAtStakePerJuror[_appeal] > pinakion.balanceOf(vote.account) ? pinakion.balanceOf(vote.account) : dispute.tokensAtStakePerJuror[_appeal];
                pinakion.transferFrom(vote.account, this, penalty);
```


## Some PoH flows (research)

### Add submission

The addSubmission function:

Example: https://etherscan.io/tx/0xf1718c5f02b376e7e0a0fb2f4b21b6e2657a258687c952fbfd50c6159c45666e

```solidity
/** @dev Make a request to add a new entry to the list. Paying the full deposit right away is not required as it can be crowdfunded later.
     *  @param _evidence A link to evidence using its URI.
     *  @param _name The name of the submitter. This parameter is for Subgraph only and it won't be used in this function.
     */
    function addSubmission(string calldata _evidence, string calldata _name) external payable {
        // Get the submissino from the sender
        Submission storage submission = submissions[msg.sender];
        // Validations from the submission 
        require(!submission.registered && submission.status == Status.None, "Wrong status");
        // Submission id
        if (submission.requests.length == 0) {
            submission.index = uint64(submissionCounter);
            submissionCounter++;
        }
        // Submission enter the vouching phase
        submission.status = Status.Vouching;
        emit AddSubmission(msg.sender, submission.requests.length);
        
        requestRegistration(msg.sender, _evidence);
    }
```

requestRegistration function:

```solidity
function requestRegistration(address _submissionID, string memory _evidence) internal {
        Submission storage submission = submissions[_submissionID];
        // if the submission has no request, the registration is the first.
        Request storage request = submission.requests[submission.requests.length++];

        // if the arbitrator changes between submission, the current request dont get stuck
        uint arbitratorDataID = arbitratorDataList.length - 1;
        request.arbitratorDataID = uint16(arbitratorDataID);

        Round storage round = request.challenges[0].rounds[0];

        // we get the arbitrator contract
        IArbitrator requestArbitrator = arbitratorDataList[arbitratorDataID].arbitrator;
        // we ask the arbitration cost from the arbitrator
        uint arbitrationCost = requestArbitrator.arbitrationCost(arbitratorDataList[arbitratorDataID].arbitratorExtraData);
        // From KlerosLiquid.sol, it means..
                /** @dev Gets the cost of arbitration in a specified subcourt.
                 *  @param _extraData Additional info about the dispute. We use it to pass the ID of the subcourt to create the dispute in (first 32 bytes) and the minimum number of jurors required (next 32 bytes).
                 *  @return cost The cost.
                 
                function arbitrationCost(bytes _extraData) public view returns(uint cost) {
                    (uint96 subcourtID, uint minJurors) = extraDataToSubcourtIDAndMinJurors(_extraData);
                    cost = courts[subcourtID].feeForJuror * minJurors;
                }
            */
        // basically, it get's the cost from feeForJuror * minJurors. court configurations. Currently, its 0.025 ETH
        // end KlerosLiquid
        uint totalCost = arbitrationCost.addCap(submissionBaseDeposit); // This is 0.1, making 0.125ETH total cost
        // round=0, request, submissioner, payed eth, value to pay 0.125. Remember this can be crowdfunded
        contribute(round, Party.Requester, msg.sender, msg.value, totalCost);
       

        // If it's full funded...
        if (round.paidFees[uint(Party.Requester)] >= totalCost)
            round.sideFunded = Party.Requester;

        // for crowfunding, there is no evidence. this event is created just once, with the video/photo upload
        if (bytes(_evidence).length > 0)
            emit Evidence(requestArbitrator, submission.requests.length - 1 + uint(_submissionID), msg.sender, _evidence);
    }
```

### Challenge request

If someone challenge request... We'll see once again the arbitrator cost being calculated but there is a createDispute now:

Example: https://etherscan.io/tx/0xfb5bae3bc14277d39ac53c769c43040c7378d8468eae2805be58b4dddb94a17b

```solidity
/** @dev Challenge the submission's request. Accept enough ETH to cover the deposit, reimburse the rest.
     *  @param _submissionID The address of the submission which request to challenge.
     *  @param _reason The reason to challenge the request. Left empty for removal requests.
     *  @param _duplicateID The address of a supposed duplicate submission. Ignored if the reason is not Duplicate.
     *  @param _evidence A link to evidence using its URI. Ignored if not provided.
     */
    function challengeRequest(address _submissionID, Reason _reason, address _duplicateID, string calldata _evidence) external payable {
        // some previous code [...]
        // ...

        Round storage round = challenge.rounds[0];
        ArbitratorData storage arbitratorData = arbitratorDataList[request.arbitratorDataID];

        uint arbitrationCost = arbitratorData.arbitrator.arbitrationCost(arbitratorData.arbitratorExtraData);
        // now, the money goes from the challenger, obviusly.
        contribute(round, Party.Challenger, msg.sender, msg.value, arbitrationCost);
        require(round.paidFees[uint(Party.Challenger)] >= arbitrationCost, "You must fully fund your side");
        round.feeRewards = round.feeRewards.subCap(arbitrationCost);
        round.sideFunded = Party.None; // Set this back to 0, since it's no longer relevant as the new round is created.
        
        challenge.disputeID = arbitratorData.arbitrator.createDispute.value(arbitrationCost)(RULING_OPTIONS, arbitratorData.arbitratorExtraData);
        // lets see the KlerosLiquid createDispute. 
        /** @dev Creates a dispute. Must be called by the arbitrable contract.
             *  @param _numberOfChoices Number of choices to choose from in the dispute to be created.
             *  @param _extraData Additional info about the dispute to be created. We use it to pass the ID of the subcourt to create the dispute in (first 32 bytes) and the minimum number of jurors required (next 32 bytes).
             *  @return disputeID The ID of the created dispute.
             */
                function createDispute(
                uint _numberOfChoices,
                bytes _extraData
                ) public payable requireArbitrationFee(_extraData) returns(uint disputeID)  {
                    (uint96 subcourtID, uint minJurors) = extraDataToSubcourtIDAndMinJurors(_extraData);
                    disputeID = disputes.length++;
                    Dispute storage dispute = disputes[disputeID];
                    dispute.subcourtID = subcourtID;
                    dispute.arbitrated = Arbitrable(msg.sender);
                    dispute.numberOfChoices = _numberOfChoices;
                    dispute.period = Period.evidence;
                    dispute.lastPeriodChange = now;
                    // As many votes that can be afforded by the provided funds.
                    dispute.votes[dispute.votes.length++].length = msg.value / courts[dispute.subcourtID].feeForJuror;
                    dispute.voteCounters[dispute.voteCounters.length++].tied = true;
                    // here is defined for the dispute how much tokens need to be staked, it wil be used then for draw a juror
                    dispute.tokensAtStakePerJuror.push((courts[dispute.subcourtID].minStake * courts[dispute.subcourtID].alpha) / ALPHA_DIVISOR);
                    dispute.totalFeesForJurors.push(msg.value);
                    dispute.votesInEachRound.push(0);
                    dispute.repartitionsInEachRound.push(0);
                    dispute.penaltiesInEachRound.push(0);
                    disputesWithoutJurors++;
                
                    emit DisputeCreation(disputeID, Arbitrable(msg.sender));
            }
        
        // end KlerosLiquid
        challenge.challenger = msg.sender;

        DisputeData storage disputeData = arbitratorDisputeIDToDisputeData[address(arbitratorData.arbitrator)][challenge.disputeID];
        disputeData.challengeID = uint96(request.lastChallengeID);
        disputeData.submissionID = _submissionID;

        request.disputed = true;
        request.nbParallelDisputes++;

        challenge.lastRoundID++;
        emit SubmissionChallenged(_submissionID, submission.requests.length - 1, disputeData.challengeID);

        request.lastChallengeID++;

        emit Dispute(
            arbitratorData.arbitrator,
            challenge.disputeID,
            submission.status == Status.PendingRegistration ? 2 * arbitratorData.metaEvidenceUpdates : 2 * arbitratorData.metaEvidenceUpdates + 1,
            submission.requests.length - 1 + uint(_submissionID)
        );

        if (bytes(_evidence).length > 0)
            emit Evidence(arbitratorData.arbitrator, submission.requests.length - 1 + uint(_submissionID), msg.sender, _evidence);
    }
```



