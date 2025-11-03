# n8n-email-follow-up
Timely follow-up is critical. This workflow takes the mental toll of tracking when to follow-up and what emails to follow-up. 

# Automating Recruiter Follow-ups: Building a Smart Email System with n8n

## The Problem

I have been applying for jobs for the last few months and I faced one challenge. I send emails to recruiters and hiring managers and then I forget about following up. Sometimes when I would go back and check it would be too soon to follow-up and sometimes it would be late by a few days.

For the ones where there are still days left to follow-up, I would tell myself to come back and follow-up, which I tend to forget. For ones where I am late by a few days, I still beat myself for not being consistent with my strategy.

I would make tasks on my calendar to follow-ups, but then when there are too many follow-up putting tasks in the calendar feels like a task as well. And, I would tell myself that I will remember this time, which I won't.

Moreover, crafting a follow-up email would require me to paste my original message to a LLM and ask it to give me a draft which I will then edit. I realise no one is anyway replying to my initial outreach and it requires 1 or 2 follow-ups to get a response. I do not go beyond 2nd follow-up because if they are not replying to 3 emails, probably they do not want to. This is something I learnt from Ace the Data Science book. The last email is hail mary.

So, I realised what if I create drafts of follow-up when I am crafting the initial outreach and just have to send it when the time comes.

## My Solution

Recently, I learnt about n8n and how people are automating their workflows and being more productive. Solving my above problem felt like a good problem to solve. I do understand it might not be a very difficult problem to solve, but I think it helps me feel organized, in control and requires less context switching when I am job hunting.

So, I looked into some workflows people have shared. And I could see that people are using LLM agents to frame emails, however I feel I do not want to outsource it to an LLM entirely. I want to vet what I am sending.

### The Workflow I Envisioned

The workflow would:
- Calculate if 3 business days (given it's also not Friday and weekend) have passed since my initial outreach. If so, I am ready to send the follow-up 1.
- Similarly, if 5 business days since Follow-Up 1 have passed and it's not Friday or Weekend then I am ready to send a Hail Mary Message.

However, I realised sending Reply to Original Message is not handled in the same way I would want to. In Gmail, when we reply as part of the same thread, it stays in the conversation. When using n8n to send emails, it creates a new email thread which has trailing email details in it. So the UI wasn't what I wanted.

This means that I can't outsource it to n8n to send those emails. I would have to send them myself. But at least having drafts and then having a system that tracks and tells you when to send the email would be helpful. I want to work inside Gmail, not move between tools.

### How Does This n8n Workflow Help Me?

It basically helps me know when it's the time to send a follow-up and I do not have to check my sent emails from time to time. If any email at any stage receives a response, it is removed from the loop.

### Why Cloud Hosting?

I initially built my workflow locally on my computer, but quickly realized I needed it running 24/7. My workflow has a scheduled trigger that checks Gmail every morning. For this to work reliably, n8n needs to be running continuously. If it's on my laptop, I'd have to remember to start it daily, and it wouldn't work if my laptop was off, asleep, or I was traveling. Cloud hosting on AWS ensures the automation runs without depending on my personal device.

## The Technical Setup

### Setting up n8n on AWS

AWS offers a free tier with 750 hours of EC2 compute per month (enough to run 24/7). Other cloud providers like Oracle Cloud also have free tiers, but I chose AWS because I already had some familiarity with it.

To set up an EC2 instance on AWS, follow a basic tutorial for launching an Ubuntu instance. The key steps are: launch an EC2 instance, configure a security group to allow port 5678, and download your SSH key pair to connect to the instance.

I used Tabby, a modern terminal with better UI/UX, to SSH into my AWS instance instead of the default terminal. It makes managing SSH connections much easier.

Once connected to the EC2 instance, I installed Docker and n8n. You can follow the official guides for Docker installation on Ubuntu and n8n Docker setup.

I ran n8n with this command to ensure my data persists:

```bash
sudo docker volume create n8n_data

sudo docker run -d --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

### Connecting to Gmail (The OAuth Challenge)

To connect n8n to Gmail, I set up Google Cloud Console credentials and enabled the Gmail API. n8n has an official guide for this setup.

When I tried to authenticate with Google, I hit an error: "OAuth callback failed because of insufficient permissions." The problem? Google requires OAuth redirect URIs to use HTTPS and a proper domain name (like https://example.com), not a public IP address like http://65.2.81.63:5678. My n8n instance was accessible via IP, but Google's security policies rejected it.

### The ngrok Solution

I found ngrok, a tunneling service that creates a temporary HTTPS domain (like https://abc123.ngrok-free.app) that forwards traffic to your server. This gave me the proper HTTPS URL that Google accepts for OAuth callbacks.

The challenge: ngrok needs to run continuously. If you close your terminal or disconnect from SSH, ngrok stops and your URL becomes inaccessible. To solve this, I used `screen`, a Linux tool that creates persistent terminal sessions. By running ngrok inside a screen session in detached mode, it continues running in the background even after I disconnect from SSH.

**Setting up ngrok with screen:**

```bash
# Install ngrok
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
sudo tar -xvzf ngrok-v3-stable-linux-amd64.tgz -C /usr/local/bin

# Add your authtoken (get this from ngrok.com after signing up)
ngrok config add-authtoken YOUR_TOKEN_HERE

# Install screen
sudo apt-get update
sudo apt-get install screen

# Start ngrok in a screen session
screen -S ngrok
ngrok http 5678

# Detach from screen (keeps ngrok running)
# Press: Ctrl+A, then D
```

**Update n8n with the ngrok URL:**

```bash
sudo docker stop n8n
sudo docker rm n8n
sudo docker run -d --name n8n \
  -e WEBHOOK_URL=https://your-ngrok-url.ngrok-free.app/ \
  -e N8N_HOST=your-ngrok-url.ngrok-free.app \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

### Building the Workflow

With n8n running on AWS and connected to Gmail, I built my workflow to check drafts, calculate business days, and send reminders.

**The Workflow Logic:**

My workflow runs every morning with a two-stage follow-up system:

**Stage 1 - First Follow-up:**
1. Find emails labeled 'Initial Outreach'
2. Filter out emails that received responses
3. Check if 3 business days passed (excluding Fridays/weekends)
4. Move qualifying emails to 'Needs Follow-up' label
5. Send me a notification email
6. After I send the drafts and approve, emails move to 'F-1 Sent' label

**Stage 2 - Final Follow-up:**
1. Check emails labeled 'F-1 Sent'
2. Filter out any that received responses
3. Check if 5 business days passed (excluding Fridays/weekends)
4. Notify me to send the final follow-up

You can download my complete workflow JSON file [here] to import directly into your n8n instance.

## Results & Lessons Learned

This system has transformed how I manage job applications. I no longer need to check my sent emails manually, remember follow-up dates, or set individual calendar reminders for each recruiter. Most importantly, drafting follow-ups while writing the initial email saves significant mental energy—I'm not starting from scratch days later when I've lost context.

The automation handles the tedious tracking work, so I can focus my energy on the actual conversations with recruiters.

### Why Not Fully Automate the Sending?

I could have built an AI agent to write and send emails automatically, but I chose not to. I want to review every message before it goes out. My writing should sound like me, not AI-generated content. This system automates the boring parts (tracking, timing, reminding) while keeping me in control of the personal touch.

### Was 2 Days of Setup Worth It?

Job hunting in this market can take months. Spending 2 days to build this automation might seem like overkill, but it's saved me daily mental energy. Every morning, I know exactly which emails need attention without manually checking or remembering. Over a long job search, that compounds significantly.

---

How does this combined version look? Any sections you'd like to adjust?
