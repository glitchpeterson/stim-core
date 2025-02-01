Smart Contract Deep Dive
Flow:

Create the coin

List the coin on the bonding curve with virtual liquidity

Buy / Sell functions post listing (to add buy fees and sell fees pre migration 1% buy ; 1% sell  )

Migrating the collected SUI liquidity and reserved virtual liquidity into a DEX LP Pool  (3% migration fees )

Functions and Structs
list - takes coinType as input and creates a pool with x SUI in virtual liquidity with 20% of token supply reserved for seeding on DEX

buy / sell - standard functions to swap on an xy = k curve with fee share configurable platform fee. Trading stops automatically when 80% of token supply is sold

make_safu - a public function to release DEX specific tickets that the caller can use to transfer the funds to that DEX’s LP pool (the function expects DEX pool object ID where the liquidity has to be migrated)

process (in kriya_adapter.move) - this function uses the ticket generated from make_safu() response to actually transfer the funds from the bonding curve pool to the DEX LP pool (the funds can’t be used for anything else or be withdrawn individually in a transaction by any admin wallet)

The platform should ideally run periodic crons to call functions 3,4 in a PTB to migrate eligible pools. Anyone can run the crons or call these functions, the function caller anyway never gets custody of the funds in their wallet via this 2 steps approach.

This method of issuing transaction specific tickets that enable trustless cross contract communication via PTB is called the hot-potato approach

Read more about it sui docs here: https://docs.sui.io/concepts/sui-move-concepts/patterns/hot-potato

Events

Bonding Curve Listing Event

Swap Event

Events tracking LP migration post bonding curve cutoff hit



to be added : 
1. a staking contract that gives out sui tokens when X token is staked 
where does this sui come from ? - it comes from the fees that is charges on buys/ sells and migration fees 

2. liquidity merger 

ontract Functionality Overview
	1.	Identifies 5 ungraduated tokens
	•	Every 24 hours, the contract checks for tokens that failed to graduate.
	•	These tokens are queued for merging.
	2.	Allows users to vote for merging
	•	Users vote using X tokens (governance or meme tokens).
	•	Votes are locked until the voting period ends.
	•	A proposal passes if it reaches minimum required votes (e.g., 10 votes).
	3.	Merges the liquidity of 5 tokens
	•	If voting succeeds and 24 hours pass, the contract removes liquidity from the original 5 tokens.
	•	The total liquidity is summed up.
	4.	Ensures proportional distribution of new tokens
	•	Before merging, the contract records each user’s share of the 5 tokens.
	•	After merging, it mints a new token and distributes it based on prior ownership percentages.
	5.	Automatically launches a new merged token
	•	A new token is created with the combined liquidity.
	•	Users receive their share based on their previous holdings.
-----------------code----------------------------
    module MemeLiquidityMerge {
    use std::signer;
    use std::vector;
    use std::option;
    use std::map;
    use std::u64;
    use std::string;
    use spot_dex;
    
    struct MergeProposal has key, store {
        creator: signer,
        tokens: vector<string::String>,
        votes: u64,
        total_weight: u64,
        timestamp: u64,
        executed: bool
    }
    
    struct VotingPool has key, store {
        voter: signer,
        staked_tokens: u64
    }
    
    struct MergedTokenPool has key, store {
        merged_token: string::String,
        liquidity: u64,
        distribution: map::Map<signer, u64>
    }
    
    public fun propose_merge(account: &signer, tokens: vector<string::String>) {
        assert!(vector::length(&tokens) == 5, 0);
        let timestamp = std::time::now_seconds();
        let proposal = MergeProposal {
            creator: signer::address_of(account),
            tokens: tokens,
            votes: 0,
            total_weight: 0,
            timestamp: timestamp,
            executed: false
        };
        let key = string::concat("proposal_", string::from_u64(timestamp));
        map::insert(&mut merge_proposals, key, proposal);
    }
    
    public fun vote_merge(account: &signer, proposal_id: string::String, stake: u64) {
        let proposal = map::borrow_mut(&mut merge_proposals, &proposal_id);
        assert!(!proposal.executed, 1);
        proposal.votes = proposal.votes + 1;
        proposal.total_weight = proposal.total_weight + stake;
        let voter_info = VotingPool { voter: signer::address_of(account), staked_tokens: stake };
        map::insert(&mut voting_pools, signer::address_of(account), voter_info);
    }
    
    public fun execute_merge(proposal_id: string::String) {
        let proposal = map::borrow_mut(&mut merge_proposals, &proposal_id);
        assert!(proposal.votes >= 10, 2); // Minimum votes required
        assert!(std::time::now_seconds() >= proposal.timestamp + 86400, 3); // 24-hour rule
        
        let merged_liquidity = 0;
        let mut distribution = map::new();
        
        for token in &proposal.tokens {
            let liquidity = spot_dex::get_liquidity(token);
            merged_liquidity = merged_liquidity + liquidity;
            spot_dex::remove_liquidity(token, liquidity);
        }
        
        for (voter, pool) in map::entries(&voting_pools) {
            let user_share = (pool.staked_tokens * 100) / proposal.total_weight;
            map::insert(&mut distribution, voter, user_share);
        }
        
        let new_token_name = string::concat("MergedToken_", string::from_u64(proposal.timestamp));
        spot_dex::create_token(new_token_name, merged_liquidity);
        
        let pool = MergedTokenPool {
            merged_token: new_token_name,
            liquidity: merged_liquidity,
            distribution: distribution
        };
        map::insert(&mut merge_pools, new_token_name, pool);
        proposal.executed = true;
    }
}




