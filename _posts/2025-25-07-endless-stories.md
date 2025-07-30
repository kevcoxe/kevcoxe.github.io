---
title: Endless Stories AI
date: 2025-07-25 11:23:00 -400
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

## Next steps
