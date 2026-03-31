The situation:

You just joined a 4-person startup. The CEO wants to launch a video streaming MVP in 6 weeks. You're the only backend engineer. Budget is tight — maybe $500/month on infra. You need something that works, not something that's perfect.

The product needs to do three things:

A user can upload a video
A user can watch a video
A user can see a list of videos

"Walk me through how you'd design the upload flow for this MVP. What happens from the moment a user hits 'Upload' to the moment the video is available to watch?"

When the user hit upload, there is an api POST request to the backend to send the full video with the header Content-Type multipart form-data and in the body, add the video file, the thumbnail file, the video title.
The video and the thumbnail are stored in an object database like S3.
The video metadata from the form and the video and thumbnail ids from the object database are stored in mongo for retreival.
the backend also to create multiple resolution for the video (4K, 1024px, 728px, 248px) and store them in S3 and their ids in mongo.
To avoid the single point of failure, we need a backend that accept requests, a microservice that process the video to create the resolutions.
And we need a Queue service like RabbitMQ so the backend can create a task in that queue then the video handler microservice can read from that queue to process the video.
After the video processing is done, the mongo db is updated.
So now the client can do a GET request to the backend to see if the video is available to watch 

# 2
Question 1 — The upload itself:
You said the client sends the full video in a single POST request to your backend.
Your MVP server is a $20/month VPS. A user uploads a 2GB video.

What happens to your backend during that upload? What if 10 people upload simultaneously?
For 10 people uploading 2GB video, the server will run out of memory.
we need to know how many users are using the app at the same time to limit the file size.
So if 10 users are using it at the same time, we can cap the file limit to 1GB per user and in the api definition, apply the cap.

You chose RabbitMQ.

RabbitMQ is a message broker. If your video processing microservice crashes mid-job and restarts — what happens to the task it was processing? Is it lost? Reprocessed? How does RabbitMQ handle that?
We can use the rabbitMQ ack to control if the message should be removed from the queue. So when the job is processed sucessfully by the consumer, the consumer can then acknowledge positively or negatively the job.

Question 3 — The "is it ready?" problem:
You said the client does a GET request to check if the video is available.

How often does the client poll? Every second? Every 5 seconds? What's the cost of that at scale, and is there a better model?
when mutiple users are uploading videos at the same time, it can overload the server.
So instead of polling, we keep the flag,  video_ready to false/true, and whe the user refresh his admin dashboard, he can get the info.
It cost less. And in the long run, if user experience is a specification, we can use the server send events to notify the client that the video is ready.


Question 4 — And the one that should make you pause:
You jumped straight to microservices, a message queue, multiple resolutions, and a separate processing service.

This is a 6-week MVP with a 4-person team and $500/month budget. Do you actually need all of this on day one? What's the cost of getting this wrong?
The cost is the deadline, and make everything work seamlessly and error handling