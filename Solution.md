Actual Flow:
Video Feed: 2 min feed input
Feed > Qwen 2.5 VL / gemini-3.1-flash-lite
Output: Key frames timestamps range. JSON.
python code: ffmpeg timestamps video feed extraction.
-------------

what is atomic reconstruction: it is the shortest description possible of the given video. the reason why it's called atomic reconstruction is because anyone reading the previous atomic reconstruction + this new atomic reconstruction should be able to fully visualize the scene / re-create their scene. That could be an LLM or a human.
Atomic reconstruction is "narrative builder". It literally builds the full scene narratively by atomically creating narration of each 2 mins indvidually by reading the previous history.

These these videos to the atomic reconstruction agent.
In parallel, send the full 2 min video to the reasoning agent.

the atomic reconstruction doesn't outputs the "timestamp" inside its final ouput "write_atomic_reconstruction()", our system literally knows for what timestamp range the model is writing that atomic reconstruction for. so our system builds the conversation history with the timestamps and atomic reconstruction. no need for model to worry about outputting timestamp.
System automatically updates the db with
video_id_timestamp: atomic_reconstruction (2 mins range)
video_id_timestamp: atomic_reconstruction (2 mins range)
In parallel:
if(no_past_atomic_reconstrunctions) then skip this. 
if(there are past atomic reconstructions) then send all those previous atomic reconstructions + last 2 mins video (uncompressed) and it will output the risk score.
So irrespective of whether suspicious activity or not, we have atomic reconstruction of the video so far.
-------------------------
narrative builder agent:
receives previous atomic reconstructions with their timestamps. 
view_key_clip(timestamp range): outputs corresponding "key clips" to that timestamp range with their atomic reconstruction.
view_key_clip(timestamp range) <can be called atleast 5 times>
// multi_modal_query() <semantic based multi-modal search. searches the all embeddings so far. even before compressing>
write_atomic_reconstruction(4-5 sentences) - exits the agent
 
reasoner agent:
receives previous atomic reconstructions + last 1 minute video stream literally (uncompressed)
view_feed_timestamp(range)
view_feed_timestamp(range) <can be called atleast 5 times>
risk_score(JSON)
-------------------------

Solving problems that matter:
what happens when both tracks aren't ready in the 2 minute? other 2 minute of the feed will pass and there will be a race and that is like a serious problem. 
we will still run the agent in parallel for that timestamp.
narrative builder agent: send all previous atomic reconstructions + "The previous timestamp range atomic reconstruction or the reasoner's decision isn't ready yet. Here is the video feed from previous timestamp key clips: {} + Here is the video feed of the current timestamp range key clips: {}, output a atomic reconstruction for the current timestamp or use tools"
reasoner agent: send all previous atomic reconstructions + "The previous timestamp range atomic reconstruction or the reasoner's decision isn't ready yet. Here is the previous timestamp feed {} + Here is the current timestamp feed: {}, output your decison or use tools" 
now this can actually again take more than 2 minute and thus again further pushing us away. and we literally allow that for 3 times. i.e. 6 minutes.
After 6 minutes, we just call these agents without providing them with view_stacked_images or view_feed_timestamp tools i.e. the same agents but only the information about these tools is removed from their system prompt.
basically, when the race condition happens and we are back 6 minutes, don't let the model use tools and just ask it to output the decision. this is more realistic because this model now has context of future from 2 min, 4 min and 6 mins.

Something like this is possible:
2 min feed processing took more than 2 min and so we are back in time.
but we started the parallel processing anyway for the 2-4 min feed.
but the first 2 min feed is ready say after 2 min 40 seconds i.e. in the middle of the processing of 2-4 min feed.
if the 2 min feed processing completes before 2 min, good. if it doesn't, let it continue
if it doesn't, as you know we start the 4-6 min processing anyway. so the critical piece of context here is that now this agent receives the 0-2 min atomic reconstruction. but doesn't receives the 2-4 min atomic reconstruction and receives it's key clips instead. this is small but nuanced context optimization. ofc, if we still didn't have first 2 min processing complete yet, then we just send it the full key feed like i mentioned earlier.
and if we are in 5th min (2-4 completed) and we still haven't received the 1st min processing, then we literally call the "tool_less" versions of those agents.
btw yes, if we are in the 5th min and we have received the first 2 min processing, then technically we have the 2 min atomic reconstruction ready and thus we only have 2 pendings now and thus we won't call tool-less agent now. Like hopefully you got the idea.


what if we are in the 5th min processing and the 2nd min processing output comes as "high risk score". well, then we literally raise the flag and the system just continues.

> Run the LLM through the LM Studio and get the endpoint URL.
> To optimize compute and speed, we implemented a dual-stream fast-slow agentic architecture
> We utilize sematic memory compression via atomic reconstructions to hierarchically caption events as they happen
> System embeds the events for the multi-modality retrieval later. Atomic reconstructions are literally multi-modal embeddings for the video in that timestamp.
> system only keeps the atomic reconstructions of the past 1 hour and they are automatically trimmed and stored in the db.
