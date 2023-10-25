# Intro: Experiments with WebArena

Notes on experimenting with [WebArena](https://github.com/web-arena-x/webarena), a web agent benchmark. See a high level overview of the paper/ benchmark [here](https://github.com/nicholaschenai/autonomous-agent-notes#webarena-a-realistic-web-environment-for-building-autonomous-agents)


## Caveats
WebArena is currently under peer review as of writing and is to be treated as experimental software. The codebase and annotations have undergone rounds of refinement over time, so the results reported below are not directly comparable if the version used is different.

- The default setting for temperature is high, so results can differ slightly under same conditions
- Bugs in annotation / evaluation / OpenAI API will lead to errors, so I reran those experiments if possible 
- The early stopping for repeating action is something that we need to tune properly. By default the param is 3 repeated actions. Some tasks might require multiple repeated actions (eg "do X action on all this user's posts") which could accidentally trigger early stop, while at the same time I do observe agents having tendencies to get stuck doing the same thing until max_steps when the repeated action early stop is lifted.

## Generic lessons when experimenting with agents
- Safeguards are important! I spotted hallucination of URLs which the agent tried to surf via the `goto` action.
- Prompting
    - Agent can confuse examples in the prompt with the task it is supposed to do, esp if there is a strong overlap (eg the maps example in WebArena occasionally causes confusion when the agent is working on maps related tasks)
    - Standardize examples in prompting: in the original prompt examples, the commands were listed with single backticks, the examples given uses triple backticks while the main parser expects triple backticks. This causes confusion to the agent and it often gives single backticks. Standardizing the number of backticks to 1 fixes this issue
    - Whenever using any form of history, need to explicitly state so in the prompt so that the agent is not confused! eg agent treating past actions as instructions and thus executing it, or past web elements which agent tries to interact with but is no longer available. By explicitly stating that historical actions are not instructions, agents get less confused, and even when they do so, they are aware that they have repeated this problem
    - instructions might not be followed: eg even when prompting agent to type answer in stop() function, it occasionally recognizes that the task is done and stops without typing the answer required by the task
    - Need to be explicit on the formatting of data given to agent -- eg without being explicit, agent doesnt actually know if the element id is on the left or right of the description
- Context limit is a challenge that we need to mitigate, occasionally exceeds in experiments when not handled properly

## WebArena-specific observations / lessons / implementation details / notes
- Truncated observation size can cause agent to be stuck in infinite loop trying to scroll down to see more
- Interference from parametric weights: On tasks related to "go to x reddit", sometimes the agent types the actual (real world) reddit link from parametric memory
- Tough example: In the gitlab webpage, if the agent surfs to a user profile, each element in the contributions chart (the tile thing where each day is colored in a shade of green representing the number of commits) will show as a line in the accessibility tree
- Reset docker + relogin before every run
- There is no password page despite the original prompt! 
- There is no task that starts with wiki page, its only a tool that is only used in map tasks
- Wikipedia env variable needs to be truncated else URL mappings will be wrong. Remove the user part
- Need a mechanism to ensure page has loaded. If the server is slow, page might not be loaded and this confuses the agent

# Experiments

Reminder: Scores cannot be compared across experiments if there are annotation / evaluation updates in between

## Default CoT in the paper
Running the CoT example in the paper to reproduce their score. I made slight modifications to the code so that the results are saved to a csv after every task, in case of errors that require me to re-run tasks. The code can be found in the next expt

WebArena Version used: 9c01d57a48bf170a24c7a50e21d881edf36d38b2 Aug 28, 2023

```bash
python run.py --instruction_path agent/prompts/jsons/p_cot_id_actree_2s.json --test_start_idx 0 --test_end_idx 812 --model gpt-3.5-turbo --result_dir results
```

### Results

| Description | Num correct | Total tasks attempted | % Performance | Comment |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Base | 50  | 812 | 6.16% | A bit lower than the paper's score of 7.14% |
| Filter out gitlab | 40  | 608 | 6.58% | Gitlab annotation had errors at that point in time so filter |

Outputs, config file and scores can be downloaded from [here](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%21119&cid=5670BDC439E69015)

### Notes
- Costs ~$25 in OpenAI credits
- Common failure mode: parsing of agent’s response fails often because of formatting issues
    - How often? 72 / 812 times (8.87%) the agent is early stopped from parsing errors (3 times in a row)! This point will be revisited in the future

## CoT + Task Chat Memory
Chat memory that persists throughout the task, but observation is limited to the latest appearance (i.e. old observations are not included), as observations are lengthy. Default CoT in the paper only has memory of the previous issued action

WebArena Version used: 9c01d57a48bf170a24c7a50e21d881edf36d38b2 Aug 28, 2023

Forked repo [here](https://github.com/nicholaschenai/webarena-memory-retrieval)

### Lessons
- Failure mode: Agent knows the full sequence of actions but since it is only allowed 1 action, it can forget the remainder of the sequence in subsequent steps. (example: it says it wants to type in the search bar, then click enter. so action 1 is type x in search bar. suprisingly, the agent did not click enter in the next step)
- While prompting agent not to repeat from history, we also need to be sure that this does not accidentally hinder future actions. I find that the agent becomes hesitant to try out old actions even if it is in a new webpage. Including URL+action in the prompt works better

### Bugs
- this expt was done with URL bug: sometimes will print unswapped URL

### Results

| Description | Num correct | Total tasks attempted | % Performance | Comment |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Base | 48  | 812 | 5.91% | - |
| Filter out gitlab | 41  | 608 | 6.74% | Gitlab annotation had errors at that point in time so filter |

Outputs, config file and scores can be downloaded from [here](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%21932&cid=5670BDC439E69015)


## CoT + VectorDB Retrieval
Retrieves the top-k relevant history to aid agent. Code works but have not fully evaluated it; Past trajectories take up a lot of context and exceeds the limit quickly — This implementation still requires a technique to condense the past results e.g. maybe another LM to summarize the history?

WebArena Version used: 9c01d57a48bf170a24c7a50e21d881edf36d38b2 Aug 28, 2023

Forked repo [here](https://github.com/nicholaschenai/webarena-memory-retrieval)

## Aside: Inspiration for methods
WebShop is a benchmark is similar to WebArena which is mature enough to have methods attempting to tackle it, so it would be good to run these methods on WebArena. Another close benchmark is MiniWob++, but it is largely solved (close to perfect accuracy)

## ReAct Prompting
Based on the concepts in [[Paper](https://arxiv.org/abs/2210.03629)]
[[Site](https://react-lm.github.io/)]
[[Code](https://github.com/ysymyth/ReAct)]

Thought-Action-Observation cycle which works on WebShop. Difference from default CoT: Refactored the code to accommodate ‘chat style’ messages and roles. Previously, it was quite monolithic with all the history being put in the user message, which can cause confusion to the agents as it mistakes the history for instruction. Originally wanted to have thought-action-observation as separate messages but this confuses the agent and it starts deviating from the action format, so the current format is to still group Thought and Action in the same message. Also, agent will do an observation summary to remind itself of the past, without exceeding context limit

WebArena Version used: 1c0f414dd7827f0eff5cbfe13cf17331f43e9793 Sep 18, 2023

Forked repo [here](https://github.com/nicholaschenai/webarena-chat-react)

### Bugs
- 4 OpenAI errors which can be fixed by re-running (2 timeout, 1 bad gateway, 1 context limit)
- 1 error from `browser_env/actions.py`, line 1552, in `create_id_based_action`, `[Unhandled Error] IndexError('list index out of range')]` in the line `else action_str.split()[0].strip()`, error when string is empty which is from the parsing. problem lies in either the LM giving improper format or parser being too rigid
- 24 `[Unhandled Error] AttributeError("'dict' object has no attribute 'split'")]` due to problem with annotation. In eval code, they expect string to split but real labels are dict

### Results

| Description | Num correct | Total tasks attempted | % Performance | Comment |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Base | 40  | 783 | 5.11% | 29 errors |

Outputs, config file and scores can be downloaded from [here](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%211400&cid=5670BDC439E69015)

## Aside: On validation

We saw how in the default CoT experiment, early stopping from parsing errors occurred for ~9% of the examples. This is significant because their baseline's accuracy is ~7%! We need a reliable way to overcome this. LangChain has structured tools, validation and fallback techniques such that:

- structured tools force the agent to only issue allowable actions (tools / functions), which can reduce syntax / parsing errors. also parsing retries / fixers
- validation in tools prevent invalid inputs, eg attempting to click on a non-existent element id
- error handling like retries when openai API fails

Other benefits include

- modularity with memory and prompts, so it is easy to do ablation studies with them
- quality of life tools like messages placeholder so user does not have to keep appending new messages to a list etc

One key downside is the difficulty in modifying default agents. Still, validation might be worth the effort. The next few experiments are done with LangChain

## LangChain Structured Tool Chat (STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION)
Difference between this and ReAct experiment from before: The latter is ReAct prompting to get the agent to emit a string, while this experiment is ReAct prompting for tool/ function selection (i.e. the action space is constrained), so parsing errors cannot happen by design (parsing errors are handled). Validation for tool inputs to ensure agent only acts on valid elements.

WebArena Version used: 58061ee914243b07756f578e03e0dc568573a7b5 Sep 28, 2023

Forked repo [here](https://github.com/nicholaschenai/webarena-langchain-agent)

### Observations
The agent's previous action occasionally confuses the agent

### Bugs
- 2 of them for task 301 and 302, annotation has wrong format. fixed in my modification and re-ran experiments
- 5 of the annotations have locators which starts with `[...`, causing errors to the evaluator which does not handle that case. This bug is fixed in WebArena release v0.2.0
- Sometimes LM generates nothing! My best guess is that the model does not see the 'Observation' world that clearly, so it tries to generate 'observation' and gets stopped (since 'observation' is a stop word)

### Results
| Description | Num correct | Total tasks attempted | % Performance | Comment |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Base | 50  | 807 | 6.20% | 5 errors |

Outputs, config file and scores can be downloaded from [here](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%212560&cid=5670BDC439E69015)

## AutoGPT
This uses the AutoGPT implementation in "Auto-GPT for Online Decision Making: Benchmarks and Additional Opinions"
[[Code](https://github.com/younghuman/LLMAgent)]
[[Paper](https://arxiv.org/abs/2306.02224)]
It is a modification of AutoGPT in LangChain for the WebShop environment. See a quick summary of the paper [here](https://github.com/nicholaschenai/autonomous-agent-notes#auto-gpt-for-online-decision-making-benchmarks-and-additional-opinions). Key components include tool use, chat memory, memory retrieval and reflection

WebArena Version used: e32b71e3f5b2463bb102457591bc06c0f2c93acf Oct 21, 2023

Forked repo [here](https://github.com/nicholaschenai/webarena-autogpt)

### Prompt breakdown

Important to manage observation length -- if the observation is too long, it will not be included in historical messages (due to the priorities in AutoGPT prompts) so the agent will not have the current observation!

| Description | Token Count, GPT 3.5-4k | Token Count, GPT 3.5-16k |
| ------------- | ------------- | ------------- |
| Total context length  | 4097  | 16,385  |
| `max_tokens` (LM token response)  | 250  | 250  |
| Base prompt (~1700 tokens) + Retrieved Memory  | 2500  |  8400  |
| Allowance for previous LM response  | 250  | 250  |
| Buffer for input msg  | 50  | 50  |
| Remainder for historical messages incl observation  | 1047  | 7435  |
| Tokens chosen for observation  | 950  | 3000 |

 GPT 3.5-4k: I initially chose 1000 tokens for the observation length, half of default 1920 used in WebArena. Midway through the experiment I reduced it to 950 as the token limit kept exceeding by a bit quite often. As the observation length is small relative to past experiments, I also wanted to see how it affects the performance. Hence I also conducted another experiment where I used GPT3.5-16k context, so that the observation length can be longer. 


### Bugs
- This is from v0.2.0 update of WebArena. `assertion error: evaluation_harness/helper_functions.py", line 172, in llm_fuzzy_match, assert "correct" in response`
    - Fuzzy match uses GPT4 to check answer. GPT4 needs to reply "correct" but it didn’t, problem is with LM

### Results

| Description | Num correct | Total tasks attempted | % Performance | Comment |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| Base 4k context | 37  | 812 | 4.56% | - |
| 16k context | 46  | 812 | 5.67% | - |

We see that context size matters! This also suggests that we can look into techniques that summarize the history so that more information can be packed into the context.

Outputs, config file, scores and message logs can be downloaded for [4k context](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%213374&cid=5670BDC439E69015) and [16k context](https://onedrive.live.com/?authkey=%21AAc0iGQGDsnrodw&id=5670BDC439E69015%215001&cid=5670BDC439E69015)

# Appendix: LangChain Notes
- structured chat output parser w retries is the default one for structured chat 
- prompt templates for structured chat agent can be found [here](https://api.python.langchain.com/en/latest/agents/langchain.agents.structured_chat.base.StructuredChatAgent.html#langchain.agents.structured_chat.base.StructuredChatAgent) and [here](https://api.python.langchain.com/en/latest/_modules/langchain/agents/structured_chat/base.html#StructuredChatAgent)
- How do agents work, step by step? [here](https://python.langchain.com/docs/modules/agents/)
- another trick to pass data to tool (useful for validation)
    - pass params to tags
    - `agent.run(input=input_text, tags=[input_text])`
    - then in tool _run method, invoke run manager.tags

# Appendix: Personal notes for WebArena v0.2.0
Changes that I think are worth taking into account/ using in new experiments

- multithreading for logins and running scripts
- retry parsing handling with max retries, but no feedback to agent
- new prompts n examples
- delete error runs at scripts/check_error_runs.py

changes which might break my custom code

- LLM calling is now separate component
- updated labels and evaluation methods, esp fuzzy matching prompt


evaluate by trace under experimental branch



# TODOs
- get visual examples from html files