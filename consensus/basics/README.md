# Basics of SA Consensus
In this section, we describe the building blocks of the SA consensus protocol, such as *blocks*, *provisioners*, *voting committees*, and *attestations*.


<!------------------------- LINKS ------------------------->
<!-- https://github.com/dusk-network/dusk-protocol/tree/main/consensus/basics/README.md -->
[vot]:  #votes
[atts]: #attestations
[att]:  #attestation
[sv]:   #stepvotes
[sr]:   #stepresult
[av]:   #aggregatevote
[pro]:  #provisioners-and-stakes
[eg]:   #extractgenerator
[ec]:   #ExtractCommittee
[cc]:   #countcredits
[sc]:   #subcommittee
[cb]:   #countsetbits
[bs]:   #bitset
[sb]:   #setbit

<!-- Consensus -->
[cenv]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#environment
[prop]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/proposal/README.md
[val]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/validation/README.md
[rat]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/ratification/README.md
[gsn]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/README.md#GetStepNum

<!-- Sortition -->
[ds]:  https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md
[dsp]: https://github.com/dusk-network/dusk-protocol/tree/main/consensus/sortition/README.md#deterministic-sortition-ds 

<!-- TODO: link to Stake Contract -->
[c-stake]: https://github.com/dusk-network/dusk-protocol/tree/main/contracts/stake

[ep]: https://github.com/dusk-network/dusk-protocol/tree/main/economic-protocol
[tok]: https://docs.dusk.network/learn/economy/tokenomics/#token-emission-schedule