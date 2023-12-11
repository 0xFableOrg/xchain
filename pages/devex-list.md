# DevEx List

A list of existing SDKs and modalities for cross-chain bridging. By necessity, this will probably be
project-per-project, and should include all steps (permissionless or not) a developer needs to
integrate the bridge.

## Non-Bridge Projects

- [Li.fi (Aggregator)](https://docs.li.fi/)
    - This has an easy to use SDK to implement with it for frontend or backend but it is unclear how the process to add an additional chain works. 

- [Interport (Aggregator)](https://docs.interport.fi/)
    - This platform seems to focus more on consumers than protocols for their integrations. It also does not provide an easy process to add additional chains so that probably wouldn't work for superchain or other environments which require permissionless adoption. 
- [Rubic (Aggregator)](https://docs.rubic.finance/)
    - This has no backend which is nice and it seems like the protocol is in charge of deploying to new chains. 
- [Squid (xchain swap on Axelar)](https://docs.widget.squidrouter.com/)
    - This relys on Axelar but I dont see any tutorials on how to implement their widget so I believe you need to contact the team to do any integrations. 
- [Socket (Liquidity Layer)](https://docs.socket.tech/)
    - This seems pretty easy to implement like Li.fi and uses multiple bridges under the hood which is nice to get more chains supported. Seems to suffer from the permissioned nature of other bridges as well. 
- [Orby (Intent Engine)](https://docs.orblabs.xyz/)
    - I like the code examples but their docs seem to be pretty lacking which is a result of the project being pretty early stage. They seem to be permissioned for new chains as well. 
- [SUAVE](https://collective.flashbots.net/c/suave/27)
    - This is tricky to discuss because I don't have clear examples of how projects used this for bridging. 