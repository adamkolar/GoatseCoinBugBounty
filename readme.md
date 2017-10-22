# GoatseCoin contract bug bounty

  * https://www.reddit.com/r/ethdev/comments/748pxi/goatsecoin_bug_bounty_our_alpha_needs_widespread/
  * Date: 2017-10-04
  * Reward: 100,000 platform tokens (should be equal to $1000)

## Contracts

[GoatseDapp.sol](./contracts/GoatseDapp.sol)

## Report

Despite what the comment in the payout function says, I think it's actually quite possible to rig the "random" selection of winners. I don't think the author accounted for the possibility that both casting of the votes and calling the finishPeriod function can happen inside a single transaction if the functions are called from a different contract.
Attacker's contract has complete information of both current vote distribution and the value of the "now" variable before casting the votes and finishing the period, he could even setup his own incentivisation scheme using part of the proceeds from the rigged lottery to ensure his transaction gets in first.

```javascript
    /* Payout 50% to winner and 50% to all voters of OC */
    function payout(string _winnerID, address _caller)
      internal
    returns (bool success)
    {
        /** 
          * Pretty weird way to get a random 10 voters but it works for now.
          * With increment and a somewhat random thing to modulo by (voteCount), it would
          * be difficult to rig beforehand by casting certain votes. What could
          * happen is a lucky miner manipulating time a second or two to rig a
          * randomWinner to be their own vote...probably not worth it.
        **/
        uint256 totalVotes;                     // Total votes; used to get a random number within vote range
        uint256 prngStart = now;
        for (uint256 i = 0; i < proposalsToday; i++) {
            totalVotes += entries[entryIDs[i]].voteCount;
        }
        
        uint256 increment = totalVotes / 10;    // Votes to pass before using voteCount to get randomWinner
        uint256 milestones;                     // Keep track of votes until increment is passed
        for (uint256 j = 0; j < proposalsToday; j++) {
            milestones += entries[entryIDs[j]].voteCount;
            if (milestones >= increment) {
                uint256 randomWinner = (prngStart * entries[entryIDs[j]].voteCount) % totalVotes;
                randomWinners.push(randomWinner);
                milestones = 0;
            }
        }
        
        assert(payVoters(randomWinners));
        assert(goatseCoin.worksIfYoureHot(entries[_winnerID].creatorAddress, 50000 * 1 ether));
        assert(goatseCoin.worksIfYoureHot(_caller, 1000 * 1 ether));
        return true;
    }
```