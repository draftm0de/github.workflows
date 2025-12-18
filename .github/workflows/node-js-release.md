workflow to run in case of MR or maybe even better for "IF MERGE READY" - I guess somehow not pressing "MERGE" , its about pressing
"Merge when ready?"

what should happen, or at least what I think
- merge (soft I guess)
- run tests again 
- build image again
- prepare next tags (if set via inputs)
- merge into target
  - tag branch with (tags)
  - tag build image and push to (in my case github)

I read about "Merge when ready" to use somehow a queue which do a "soft merge, run .... and if completed than merge"
maybe these workflow has two "trigger?!"