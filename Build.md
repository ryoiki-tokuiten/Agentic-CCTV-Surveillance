Actual Flow:
Video Feed: 2 min feed input
Feed > gemini-3.1-flash-lite
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
narrative builder agent: gemini-3.1-flash-lite too.
receives previous atomic reconstructions with their timestamps. 
view_key_clip(timestamp range): outputs corresponding "key clips" to that timestamp range with their atomic reconstruction.
view_key_clip(timestamp range) <can be called atleast 5 times>
write_atomic_reconstruction(4-5 sentences) - exits the agent
 
reasoner agent: gemini-3.1-flash-lite too.
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
---------------------------


Embed the atomic reconstructions with the video feed of that timestamp. Like literally bro do you understand we have text embedding corresponding to that video. It is extremely useful for semantic based multi-modal search. searches the all embeddings so far. Use gemini multi-modal embedding for this through API. It's completely free.


Fully implemet this.  Make sure you don't change any logic or any decision i have taken. This is absolutey crucial and must.

UI matters a lot for this. Use react, tailwind css and external scripts and libraries for beautifully representing this. 

    Panel 1 (Live): The video feed.

    Panel 2 (Brain): Show the full agent activitiy of the atomic reconstruction agent. Scroll the text of the Atomic Reconstructions as they are generated.

    Panel 3 (Alerts): Flash red when the risk scores are high.

    Panel 4 (Verifier): Show the specific clip being analyzed by the Slow Agent with a "Analyzing..." spinner.
    
    ----------------------
    
    
    Allow me to upload videos literally inside the browser. Make sure that i literally see in the dashboard what feed is being currently processed. Beautifully show when the race condition is happened and how it's being handled. Show the UI
