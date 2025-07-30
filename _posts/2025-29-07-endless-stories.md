---
title: Endless Stories AI
date: 2025-07-29 09:00:00 -400
author: kevcoxe
categories: [Blog, Projects]
tags: [ai, nextjs, aws, devops]
---

Have you ever thought of a product or project and wanted to make it but then were nervous that you might fail?
This is how I feel quite often, I have wanted to build some public facing site and just been to nervous that it might fail to do it.
But this is a story of the time I created my first public software as a service site.

# What is Endless Stories?

## Inspiration
My kids had been listening to a few audio books and stories and one day my son asked if he could make his own book.
This was a cool idea, but the issue was he did not know how to spell or write.

I wanted to help him solve this issue and make the thing he wanted.
I opened my notes app and turned my phone to speach to text.
He sat down and I left him alone to speak out his story idea.
When he was done he showed it to me, there were a bunch of scrambled ideas, but there was a plot.
I asked if he wanted help making it a longer story, and he said yes.

Next I took his story idea and prompted [ChatGPT](https://chatgpt.com/), I asked to turn his notes into a structured outline for a few chapters of a story.
A few seconds later I had a nice outline of a story.
I then took each chapter outline and asked to expand on it to be about 1000 words or so, but to make sure to keep the underlying story.

Next we needed a voice to read out his story since well he could not read.
I headed over to [Eleven Labs](https://elevenlabs.io/) and looked for a voice that sounded like a pirate (his story was about pirates).
I then had the AI voice read out each of his chapters and saved them.
I loaded them onto his [Yoto player](https://us.yotoplay.com/) so he could enjoy his new audio book.

My son loved his book, he sent it to all of his friends and family.
He was then inspired to make more and more stories.

I loved that he was able to make something and it helped spark imagination to build more.
I wanted to find a way to allow others do the same.
I thought why not make a website that automated the process allowing others to create stories of their own.

I created [Endless Stories](https://endlessstories.ai).
After giving a basic story idea from a few words to many paragraphs, you could select the desired output length of the story and a narator to read your story.
The app would then generate the rest.
It would generate the story keeping your idea in mind, expanding it.
It would create a title for the story, audio naration, and a cover picture for the story.
You would get an email when your story was ready with a link to your story.
You were able to download your story for offline use and be able to use it wherever you wanted.

Along with the stories you created you could see the full catalog of all of the stories made by others, you could read or listen to whatever you wanted.
The idea was to help inspire you to create your own stories.
There were stories from about the real reason why the chicken crossed the road, to a world where everyone was named Kevin, to a chilling quest in a dark castle.

![chicken-cross-the-road](/assets/img/posts/endless-stories/story_4b64c4ee-26b2-49be-88f6-0dfcb072732b.webp){: width="200" height="200" .normal }
![castle](/assets/img/posts/endless-stories/story_8.webp){: width="200" height="200" .normal }
![dragon](/assets/img/posts/endless-stories/story_5.webp){: width="200" height="200" .normal }

My kids loved making stories, they would input a few ideas and love to hear what would be created for them, we created hundreds of stories.

## Design process

For designing the app I knew I wanted to use [Next.js](https://nextjs.org/) and [Supabase](https://supabase.com/) because that was what I was comfortable with.
I knew I would need a place to deploy my Next.js app to and then I was going to use managed Supabase.
As for the text/image/audio generation I decided to go with [OpenAI API](https://openai.com/api/) because it was super easy to use and I liked the results.

## Development

### Phase 1 MVP
In about 2 days I had a working prototype where I had a home page with a list of all the stories and you could read them and see the pictures.
I had a form to input your story idea and it would generate everything.

This was exciting to see, but there were issues.
- Creating all of the parts to the story in one API call took quite a long time
- I was using supabase for all of the storage, text, image, audio, it was going to get expensive

### Phase 2 Move to AWS
I like to use [Terraform](https://www.hashicorp.com/en/products/terraform) when I deploy anything to AWS.
So first thing I did was create my S3 bucket in AWS that would story my Terraform state.
I like to do this so that way when (not if, but when) my computer dies I have the state of my infrastructure saved to the cloud.

Next I created a S3 bucket to for my asset storage and IAM user for the app to be able to access the S3 bucket.

#### Storing Assets
First thing I did was move my storage to AWS, I was still keeping supabase for the main database to story info on each story, but I moved all images and audio files to AWS S3.
S3 was cheap and easy to use. This can come with a downside tho.
You do not want to have an open S3 bucket that is visible to the world that anyone can access, so how would I be able to access those files from my Next.js app?

The answer is [Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html).
This would generate a signed URL that could have an expiration so it would be valid while the client tried to download the asset but then after let's say 60 seconds be invalid not letting others view the asset.

#### Fixing Long Calls to AI
There are some general things you need to be able to handle when dealing with calls to an AI API.
- The long time it can take for a response
- Getting rate limited
- API errors
- Dealing with bogus data

When you start to chain multiple requests behind your one API call things will start to have issues.
Yes you can add some status or break up your calls into multiple calls, but still if you have a set number of tasks that need to happen why not use a message queue.

For this I went with an AWS SQS with AWS lambda function pipeline.
The basic idea is you would put a message onto your message queue and then a lambda function will pick up that message processing it.
If a message fails, you can have a set number of retries and a delay, this is great for dealing with the rate limiting or API errors.
If a message fails past that, you can add the message to a dead letter queue and have a seperate lambda function deal with them.

![aws pipeline](/assets/img/posts/endless-stories/endless_stories-sqs-lambda-workflow.png)

There is an issue where you might not want your lambda function running to long and some of these AI calls can take a while, but I did not run into any issues here.

I decided to go with one lambda function that based on the message state would do different tasks.

- Moderate input story
- Generate story
- Moderate final story
- Generate title
- Generate image prompt
- Generate image
- Generate audio
- Generate tags
- Generate metadata
- Notify user

![aws sqs order](/assets/img/posts/endless-stories/sqs-message-order.png)


### Phase 3 Ditching Vercel
Originally I had deployed to [Vercel](https://vercel.com/), they make Next.js and make deploying there insanely easy.
I do not mind deploying there, but I did have a few issues.
I was using an organization on [Github](https://github.com/) and Vercel wanted me to also pay to be able to use that repo.
I was trying to reduce the number of services I was paying for for this app that I had no idea if it would even make money.
I also wanted to try out [AWS Amplify](https://aws.amazon.com/amplify/).

Amplify was easy to setup, you can connect your github repo to it and when you push code it will auto deploy your project.
I also liked having most of my expenses in one place.
AWS also has a pretty generous free tier, so those "expenses" were mostly S3 storage.

### Phase 4 Adding Payments
I did not want to deploy this site and it become an overnight success and have no way of getting paid.
I also wanted to learn about using [Stripe Payments](https://stripe.com/payments).

Adding Stripe integration was very easy, there are many tutorials out there and I had found one that integrated with Supabase.
This took me about a day of work.
Now I have a way of adding "Credits" to a user and you spend your credits creating stories.
I had the idea of maybe having multiple price options or using a premium voice option, but I never did.


## Final Thoughts
I really enjoyed creating this site. I used it multiple times a day.
I learned a lot, from dealing with AI API calls to adding payments for a service.
In the end I was the top user of the site, but I also would do it again in a heartbeat.

The status of the site right now is that it is not running. Like I said I was trying to not be paying for a bunch of services when I did not make any money off of it.
So I used managed Supabase which has a free tier, this is great.
One of the limitations of the free tier is that if your project is not used after 7 days it will be paused.

I could just keep using the site every couple days, but I moved onto my next project and did not have time.
I am working on moving this to a self hosted model so I do not need to use AWS or managed Supabase.
I will keep you posted.


<!-- 
Now I want to talk about how I created this app and some of the problems I faced and how I resolved them.

I knew I wanted to use [Next.js](https://nextjs.org/) since it was easy to spin up a webapp and I enjoyed the server actions portion which made calling backend functions easy.
So I had a framework, next I needed some way of storing the data for the site. My goto at the time was [Supabase](https://supabase.com/).
I love Supabase because it is very easy to start and it includes a lot of features like authentication.
Now I had used [ChatGPT](https://chatgpt.com/) and [Eleven Labs](https://elevenlabs.io/) before for my sons's story, but for this I would need to use the [OpenAI API](https://openai.com/api/) which is the api behind ChatGPT.
I checked out the OpenAI api and saw I could do more than just text, I could also do text to speach and text to image. This was great.

The stack I decided to use was
- [Next.js](https://nextjs.org/)
- [Supabase](https://supabase.com/)
- [OpenAI](https://openai.com/api/)

I soon had a simple site that I could log into and enter a story idea, I would wait for the call to OpenAI to finish and see the story output.
This was great, but this also showed an issue.
I had quite a few steps to have all of the items generated.
Here are the requests I made.

1. Generate the full story
2. Generate a title from the full story
3. Generate a prompt for the cover image
4. Generate the cover image
5. Generate the naration

That is quite a few requests to OpenAI, there are a few issues that can come up because these are long running requests.
The first issue is that since it takes so long it makes your site look un-responsive.
I tried to fix this by showing a progress bar saying what step it was on, but it still was not a good fix.

I was watching a bunch of people on YouTube on Next.js and other web dev stuff.
Someone mentioned a product that worked with Next.js and allowed you to create a queue of tasks specifically for AI calls.
I loved it, but it had a price tag and I didnt want to keep signing up for a bunch of services when I could just do it myself.
I had experience working with queues in AWS and so I decided to do it myself.

I decided that I would create a little [AWS Simple Queue Service](https://aws.amazon.com/sqs/) with [AWS Lambda](https://aws.amazon.com/lambda/) pipeline.
I decided to manage all of my AWS infrastructure using [Terraform](https://www.hashicorp.com/en/products/terraform), this allowed me to define my infrastrucre as code to easily keep track of everything I had deployed.
This is my goto for whenever I need to deploy any AWS resources.

First I created a [AWS S3 bucket](https://aws.amazon.com/s3/) to store the state for my Terraform.
Next I created two SQS queues
- One queue for the messages
- One for the dead letter queue, for when the message fails

The idea I had was when a user would create a story, it would add a message with some info and which step I am on.
- Create story
- Create title
- Create image prompt
- Create image
- Create naration
- Combine audio and image into one file
- Verify story is appropriate
- Notify the user

When a message was added to the SQS queue a Lambda functionw as spun up to handle the message, I also had a limit in place to only spin up to 20 Lambdas at a time.
Each Lambda was the same and depending on the state of the message the Lambda would do some specific function.
Once a message was sucessfully processed it would be removed from the queue, if a message failed from an error or if one of the api calls timed it it would be re added to the queue.
When the last message was finished for a story it would send an email update to the user to let them know their story was ready and have a link to the story with a download link as well.
There was a retry limit of 5, if a message fails 5 times it would be added to the dead letter queue.
There was also a Lambda that would processs those messages, I would recieve an alert that a message failed, I would credit back the user, and send them an apology email.

![aws pipeline](/assets/img/posts/endless-stories/endless_stories-sqs-lambda-workflow.png)

### Initial design

### Problems I had to solve

### Final version

## Next steps -->
