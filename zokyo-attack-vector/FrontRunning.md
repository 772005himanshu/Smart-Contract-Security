## Front-Running 

Smart contract have revolutionized the way transaction take place in blockchain ecosystem by providing trustless and automated solution

## Understanding Front-Running
For understanding front-running , we need to grasp how the transaction work in the ethereum blockchain


### Understanding Gas Price

1. Transaction lifecycle in Ethereum
When user call the transaction it goes to the mempool , there is sort of the waiting room for all pending transaction , the transaction waits until a miner picks it up. 

- Miners select transactions from the mempool to add to the new block. Primarily, they prioritize transactions offering higher gas prices

### Front Running in Action
- Front running exploits the transparent and deterministic nature of transaction execution in Ethereum.

- The attacker monitors the mempool for potentially profitable transaction.eg. a transaction where a user is about to purchase a rare token in a DEX

- Upon spotting such a transaction , the attacker crafts a similar transaction , they set higher gas price for their transaction

- Miner priortize the transaction with higher gas prices , They are likely to select the attacker's transaction before the original one

- If everything goes as planned, the attacker's transaction will be confirmed first, allowing them to reap the benefits intended for the original sender


## Example 

### Simple Example by Solidity by example

```solidity
contract FindThisHash {
    bytes32 public constant hash =
        0x564ccaf7594d66b1eaaea24fe01f0585bf52ee70852af4eac0cc4b04711cd0e2;

    constructor() payable {}

    function solve(string memory solution) public {
        require(hash == keccak256(abi.encodePacked(solution)), "Incorrect answer");

        (bool sent, ) = msg.sender.call{value: 10 ether}("");
        require(sent, "Failed to send Ether");
    }  // Before the user get rewards attacker see this transaction and get the solution from the transaction and get same solution submit the trasanction with higher gas to validate first 
}
```

#### code4rena Buf find with medium

```solidity
    /// @dev Create a new vault
    function build(address owner, bytes12 vaultId, bytes6 seriesId, bytes6 ilkId)
        external
        auth
        returns(DataTypes.Vault memory vault)
    {
        require (vaultId != bytes12(0), "Vault id is zero");
        require (vaults[vaultId].seriesId == bytes6(0), "Vault already exists");   // Series can't take bytes6(0) as their id
        require (ilks[seriesId][ilkId] == true, "Ilk not added to series");
        vault = DataTypes.Vault({
            owner: owner,
            seriesId: seriesId,
            ilkId: ilkId
        });
        vaults[vaultId] = vault;

        emit VaultBuilt(vaultId, owner, seriesId, ilkId);
    }  // you get the vaultID before form the transaction and submit it before the user , it doesnot profit but cannot open vault for user in the protocol , by repeating the same process we can stop all user from using the vault
```

Medium-severity impact: While the likelihood of this may be low, the impact is high because valid vaults will never be successfully created and will lead to a DoS against the entire protocol’s functioning. So, with low likelihood and high impact, the severity (according to OWASP) is medium.

#### Real life Cod4rena Bug Find With Payout (High Risk Vulnerability)


```solidity
// Calculate output of swapping SPARTA for TOKEN & update recorded amounts
    function _swapBaseToToken(uint256 _x) internal returns (uint256 _y, uint256 _fee){
        uint256 _X = baseAmount;
        uint256 _Y = tokenAmount;
        _y =  iUTILS(_DAO().UTILS()).calcSwapOutput(_x, _X, _Y); // Calc TOKEN output
        uint fee = iUTILS(_DAO().UTILS()).calcSwapFee(_x, _X, _Y); // Calc TOKEN fee
        _fee = iUTILS(_DAO().UTILS()).calcSpotValueInBase(TOKEN, fee); // Convert TOKEN fee to SPARTA
        _setPoolAmounts(_X + _x, _Y - _y); // Update recorded BASE and TOKEN amounts
        _addPoolMetrics(_fee); // Add slip fee to the revenue metrics
        return (_y, _fee);
    }
```

The Function does not implement any slippage checks with comparing the swap / liquidity results with a minimum swap / liquidity value

Users can be frontrun and receive a worse price than expected when they initially submitted the transaction. There's no protection at all, no minimum return amount or deadline for the trade transaction to be valid which means the trade can be delayed by miners or users congesting the network, as well as being sandwich attacked - ultimately leading to loss of user funds

#### $2500 payout , Medium Risk

```solidity
 function bondForWithHint(
        uint256 _amount,
        address _owner,
        address _to,  // making the user transcoder by front-running
        address _oldDelegateNewPosPrev,
        address _oldDelegateNewPosNext,
        address _currDelegateNewPosPrev,
        address _currDelegateNewPosNext
    ) public whenSystemNotPaused currentRoundInitialized {
        // the `autoClaimEarnings` modifier has been replaced with its internal function as a `Stack too deep` error work-around
        _autoClaimEarnings(_owner);
        Delegator storage del = delegators[_owner];

        uint256 currentRound = roundsManager().currentRound();
        // Amount to delegate
        uint256 delegationAmount = _amount;
        // Current delegate
        address currentDelegate = del.delegateAddress;
        // Current bonded amount
        uint256 currentBondedAmount = del.bondedAmount;

        if (delegatorStatus(_owner) == DelegatorStatus.Unbonded) {
            // New delegate
            // Set start round
            // Don't set start round if delegator is in pending state because the start round would not change
            del.startRound = currentRound.add(1);
            // Unbonded state = no existing delegate and no bonded stake
            // Thus, delegation amount = provided amount
        } else if (currentBondedAmount > 0 && currentDelegate != _to) {
            // Prevents third-party caller to change the delegate of a delegator
            require(msg.sender == _owner || msg.sender == l2Migrator(), "INVALID_CALLER");
            // A registered transcoder cannot delegate its bonded stake toward another address
            // because it can only be delegated toward itself
            // In the future, if delegation towards another registered transcoder as an already
            // registered transcoder becomes useful (i.e. for transitive delegation), this restriction
            // could be removed
            require(!isRegisteredTranscoder(_owner), "registered transcoders can't delegate towards other addresses");
            // Changing delegate
            // Set start round
            del.startRound = currentRound.add(1);
            // Update amount to delegate with previous delegation amount
            delegationAmount = delegationAmount.add(currentBondedAmount);

            decreaseTotalStake(currentDelegate, currentBondedAmount, _oldDelegateNewPosPrev, _oldDelegateNewPosNext);
        }

        {
            Transcoder storage newDelegate = transcoders[_to];
            EarningsPool.Data storage currPool = newDelegate.earningsPoolPerRound[currentRound];
            if (currPool.cumulativeRewardFactor == 0) {
                currPool.cumulativeRewardFactor = cumulativeFactorsPool(newDelegate, newDelegate.lastRewardRound)
                    .cumulativeRewardFactor;
            }
            if (currPool.cumulativeFeeFactor == 0) {
                currPool.cumulativeFeeFactor = cumulativeFactorsPool(newDelegate, newDelegate.lastFeeRound)
                    .cumulativeFeeFactor;
            }
        }

        // cannot delegate to someone without having bonded stake
        require(delegationAmount > 0, "delegation amount must be greater than 0");
        // Update delegate
        del.delegateAddress = _to;
        // Update bonded amount
        del.bondedAmount = currentBondedAmount.add(_amount);

        increaseTotalStake(_to, delegationAmount, _currDelegateNewPosPrev, _currDelegateNewPosNext);

        if (_amount > 0) {
            // Transfer the LPT to the Minter
            livepeerToken().transferFrom(msg.sender, address(minter()), _amount);
        }

        emit Bond(_to, currentDelegate, _owner, _amount, del.bondedAmount);
    }

```

In the current implementation of the bondForWithHint function in the provided smart contract, there is a potential for a front-running attack that may force a user to unintentionally become a transcoder, preventing them from delegating their tokens to other users

- This can be achieved by an attacker front-running a transaction where a user (let's call this user 'Adam') is trying to delegate tokens to another user (let's say, 'Bob') using the bondWithHint (or bondForWithHint) function. The attacker can front-run Adam's transaction by calling bondForWithHint, setting _owner to Adam's address, _to to Adam's address again, and the bond amount to any non-zero value (even as low as 1 wei

The malicious transaction is coded as follows:
```solidity
bondForWithHint(1, Adam, Adam, /*other parameters*/);
```


### Bug Bounty Payout &20000 High risk

```solidity
  
      struct Report {
        address reporter;
        address reportedAddress;
        address secondReportedAddress;
        uint256 reportTimestamps;
        ILERC20 reportTokens;
        bool secondReports;
        bool reporterClaimStatus;
    }
  
  
  /// @notice This function will generate a report
    /// @dev This function must be called by a non blacklisted/reported address. 
    /// It will generate a report for an address linked to a token.
    /// Lossless Contracts, Admin addresses and Dexes cannot be reported.
    /// @param _token Token address of the stolen funds
    /// @param _account Potential malicious address
    function report(ILERC20 _token, address _account) override public notBlacklisted whenNotPaused returns (uint256){
        require(_account != address(0), "LSS: Cannot report zero address");
        require(!losslessController.whitelist(_account), "LSS: Cannot report LSS protocol");
        require(!losslessController.dexList(_account), "LSS: Cannot report Dex");

        uint256 reportId = tokenReports[_token].reports[_account];

        require(reportId == 0 || 
                reportInfo[reportId].reportTimestamps + reportLifetime < block.timestamp || 
                losslessGovernance.isReportSolved(reportId) && 
                !losslessGovernance.reportResolution(reportId), "LSS: Report already exists");

        reportCount += 1;
        reportId = reportCount;
        reportInfo[reportId].reporter = msg.sender;

        tokenReports[_token].reports[_account] = reportId;
        reportInfo[reportId].reportTimestamps = block.timestamp;
        reportInfo[reportId].reportTokens = _token;

        require(stakingToken.transferFrom(msg.sender, address(this), reportingAmount), "LSS: Reporting stake failed");

        losslessController.addToBlacklist(_account);
        reportInfo[reportId].reportedAddress = _account;
        
        losslessController.activateEmergency(_token);

        emit ReportSubmission(_token, _account, reportId, reportingAmount);

        return reportId;
    }


    /// @notice This function will add a second address to a given report.
    /// @dev This funtion must be called by a non blacklisted/reported address. 
    /// It will generate a second report linked to the first one created. 
    /// This can be used in the event that the malicious actor is able to frontrun the first report by swapping the tokens or transfering.
    /// @param _reportId Report that was previously generated.
    /// @param _account Potential malicious address
    function secondReport(uint256 _reportId, address _account) override public whenNotPaused {
        require(_account != address(0), "LSS: Cannot report zero address");
        require(!losslessGovernance.isReportSolved(_reportId) && !losslessGovernance.reportResolution(_reportId), "LSS: Report already solved");
        require(!losslessController.whitelist(_account), "LSS: Cannot report LSS protocol");
        require(!losslessController.dexList(_account), "LSS: Cannot report Dex");

        Report storage queriedReport = reportInfo[_reportId]; 

        uint256 reportTimestamp = queriedReport.reportTimestamps;
        ILERC20 token = queriedReport.reportTokens;

        require(_reportId != 0 && reportTimestamp + reportLifetime > block.timestamp, "LSS: report does not exists");
        require(queriedReport.secondReports == false, "LSS: Another already submitted");
        require(msg.sender == queriedReport.reporter, "LSS: invalid reporter");

        queriedReport.secondReports = true;
        tokenReports[token].reports[_account] = _reportId;

        losslessController.addToBlacklist(_account);
        queriedReport.secondReportedAddress = _account;

        emit SecondReportSubmission(token, _account, _reportId);
    }
    
    /// @notice This function solves a report based on the voting resolution of the three pilars
    /// @dev Only can be run by the three pilars.
    /// When the report gets resolved, if it's resolved negatively, the reported address gets removed from the blacklist
    /// If the report is solved positively, the funds of the reported account get retrieved in order to be distributed among stakers and the reporter.
    /// @param _reportId Report to be resolved
    function resolveReport(uint256 _reportId) override public whenNotPaused {

        require(!isReportSolved(_reportId), "LSS: Report already solved");


        (,,,uint256 reportTimestamps,,,) = losslessReporting.getReportInfo(_reportId);
        
        if (reportTimestamps + losslessReporting.reportLifetime() > block.timestamp) {
            _resolveActive(_reportId);
        } else {
            _resolveExpired(_reportId);  // reading the report ID from here get submit it before anyone else 
        }
        
        reportVotes[_reportId].resolved = true;
        delete reportedAddresses;  // reported address from read it go to first place front run it  // this address is used as _account in sedond report funtion and first function

        emit ReportResolve(_reportId, reportVotes[_reportId].resolution);
    }
    
    /// @notice This function is for the reporter to claim their rewards
    /// @param _reportId Staked report
    function reporterClaim(uint256 _reportId) override public whenNotPaused {
        require(reportInfo[_reportId].reporter == msg.sender, "LSS: Only reporter");
        require(losslessGovernance.reportResolution(_reportId), "LSS: Report solved negatively");

        Report storage queriedReport = reportInfo[_reportId];

        require(!queriedReport.reporterClaimStatus, "LSS: You already claimed");

        queriedReport.reporterClaimStatus = true;

        uint256 amountToClaim = reporterClaimableAmount(_reportId);

        require(queriedReport.reportTokens.transfer(msg.sender, amountToClaim), "LSS: Token transfer failed");
        require(stakingToken.transfer(msg.sender, reportingAmount), "LSS: Reporting stake failed");
        emit ReporterClaim(msg.sender, _reportId, amountToClaim);
    }


```
- but its not the reporter being front-run....its the reporter doing the front-running! 

- This is because after a report is voted on by the governance and validated, when the governance calls resolveReport, the reporter can listen to this contract call, and front run this transaction by calling secondReport and inputting the address of a victim (e.g. a whale or a dex). Because this transaction is completed before 'resolveRepot', 'resloveReport' now pulls the tokens from the victim and distributes it to everybody involved, the proposedWallet, stakers, community members.

## Mitigation Steps for Auditing

- Commit-Reveal Solution

```solidity
contract SecuredFindThisHash {
    // Struct is used to store the commit details
    struct Commit {
        bytes32 solutionHash;
        uint commitTime;
        bool revealed;
    }

    // The hash that is needed to be solved
    bytes32 public hash =
        0x564ccaf7594d66b1eaaea24fe01f0585bf52ee70852af4eac0cc4b04711cd0e2;

    // Address of the winner
    address public winner;

    // Price to be rewarded
    uint public reward;

    // Status of game
    bool public ended;

    // Mapping to store the commit details with address
    mapping(address => Commit) commits;

    // Modifier to check if the game is active
    modifier gameActive() {
        require(!ended, "Already ended");
        _;
    }

    constructor() payable {
        reward = msg.value;
    }

    /* 
       Commit function to store the hash calculated using keccak256(address in lowercase + solution + secret). 
       Users can only commit once and if the game is active.
    */
    function commitSolution(bytes32 _solutionHash) public gameActive {
        Commit storage commit = commits[msg.sender];
        require(commit.commitTime == 0, "Already committed");
        commit.solutionHash = _solutionHash;
        commit.commitTime = block.timestamp;
        commit.revealed = false;
    }

    /* 
        Function to get the commit details. It returns a tuple of (solutionHash, commitTime, revealStatus);  
        Users can get solution only if the game is active and they have committed a solutionHash
    */
    function getMySolution() public view gameActive returns (bytes32, uint, bool) {
        Commit storage commit = commits[msg.sender];
        require(commit.commitTime != 0, "Not committed yet");
        return (commit.solutionHash, commit.commitTime, commit.revealed);
    }

    /* 
        Function to reveal the commit and get the reward. 
        Users can get reveal solution only if the game is active and they have committed a solutionHash before this block and not revealed yet.
        It generates an keccak256(msg.sender + solution + secret) and checks it with the previously commited hash.  
        Front runners will not be able to pass this check since the msg.sender is different.
        Then the actual solution is checked using keccak256(solution), if the solution matches, the winner is declared, 
        the game is ended and the reward amount is sent to the winner.
    */
    function revealSolution(
        string memory _solution,
        string memory _secret
    ) public gameActive {
        Commit storage commit = commits[msg.sender];
        require(commit.commitTime != 0, "Not committed yet");
        require(commit.commitTime < block.timestamp, "Cannot reveal in the same block");
        require(!commit.revealed, "Already commited and revealed");

        bytes32 solutionHash = keccak256(
            abi.encodePacked(Strings.toHexString(msg.sender), _solution, _secret)
        );
        require(solutionHash == commit.solutionHash, "Hash doesn't match");

        require(keccak256(abi.encodePacked(_solution)) == hash, "Incorrect answer");

        winner = msg.sender;
        ended = true;

        (bool sent, ) = payable(msg.sender).call{value: reward}("");
        if (!sent) {
            winner = address(0);
            ended = false;
            revert("Failed to send ether.");
        }
    }
}
```