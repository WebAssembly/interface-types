# Working Notes

The purpose of this directory is to hold notes about the current design of the
Interface Types proposal. These notes are intended as analogous to the
[WebAssembly design repo](https://github.com/WebAssembly/design/), as a
less-formal way of designing components of the proposal. Documents should
reflect our current consensus, but the ideas don't need to be fully fleshed out.
These documents should inform the final spec.

Sample topics:
* How we should handle Wasm exceptions
* How engines might need to optimize combinations of adapters
* What is necessary for MVP, and what can be deferred

In general, we should try to match the conventions that are already established,
but when inventing some new topic, just making up syntax/instructions is the
right way to go.

### Q: Why not add these to the Explainer?
These aren't necessarily things that the explainer needs to spell out. If the
purpose of the explainer is to convey the information to a reader from scratch,
the nuances of a given design detail is likely to be distracting detail. It is
likely that some subsets will wind up in the explainer over time.
