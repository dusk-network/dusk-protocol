# Review Process Guidelines
The goal of a review process is to correctly and thoroughly document a part of the protocol.

This document describes the pattern to follow to review a single protocol component.

## Review Process Steps
> NOTE: The word _item_ indicates the component to review.

 1. **Collect information**: search existing documentation and code related to _item_.
<!-- info can be copied in the NOTES.md file  -->

 2. **Index information**: add all sources (documents and code files) found in Step 1
                           to the _keyword index_, under the voice _item_.

 3. **Review code**: read related code, review documentation, and identify code to be discussed.

 4. **Update documentation**: use all gathered information to write a complete documentation.

 5. **Start research**: open Issues to clarify lacking rationale, address existing issues, and research for improvements.

---

 ### Step 1: Collect information
 Information on the Dusk protocol should be searched through all these sources:
 - [Whitepaper](https://dusk.network/uploads/The_Dusk_Network_Whitepaper_v3_0_0.pdf)
 - [GitBook](https://app.gitbook.com/o/-M8H04Dfb0IChBw1yUbU/home) (until migration to Wiki is completed)
 - [Dusk Wiki](https://wiki.dusk.network)
 - [Miro boards](https://miro.com/app/dashboard/)
 - [GitHub repositories](https://github.com/orgs/dusk-network/repositories) (READMEs, code, Issues, ...)

### Step 2: Index information
All sources found in Step 1 should be added to the [_keyword index_](https://github.com/dusk-network/dusk-index/tree/main/keywords).
Sources are divided into _Docs_ and _Code_. Docs should include links to all textual documents (Wiki, READMEs, etc...). Code should list all related repos, source files, and main functions/structures.

### Step 3: Review code
This phase should process each repo and code listed in the _Code_ section of the index build in Step 2.

Code review should be split in different items:
- **Document code**: comments in the code should be updated and clarify what the code does in detail.
- **Discuss**: if some code is unclear (e.g., it's hard to tell WHY something is done), affects security/performance, or has inconsistencies with the specification, an Issue should be created (in the same repo). Issues should be _atomic_, that is, address a single discussion item. 
- **Create research tasks**: when a research question arises, a new Issue should be open in [`dusk-protocol`](https://github.com/dusk-network/dusk-protocol/) describing the question and its possible implications.