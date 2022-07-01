# Some notes about how PoH uses an arbitrator

### Add submission

The addSubmission function:

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

If someone challenge request.. We'll see once again the arbitrator cost being calculated but there is a createDispute now:

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



