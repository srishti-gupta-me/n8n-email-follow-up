# n8n-email-follow-up
Automated workflow for tracking and managing email follow-ups during job search. Uses n8n hosted on AWS to monitor outreach emails and send reminders when it's time to follow up.

## Features
- Tracks emails labeled 'Initial Outreach' and 'F-1 Sent'
- Calculates business days (excludes weekends and Fridays)
- Filters out emails that received responses
- Sends email notifications if any follow-ups are needed
- Two-stage follow-up system (3 days for first follow-up, then 5 days for final follow-up)

## Prerequisites
- AWS account (free tier eligible)
- Google Cloud Console account (free tier)
- ngrok account (free tier)
- Basic familiarity with SSH and Docker (learn on the go)

## Installation

### 1. Set up EC2 Instance
Launch an Ubuntu EC2 instance on AWS, configure security group to allow port 5678 (n8n uses this), and SSH into it.

### 2. Install Docker and n8n on the EC2 Instance

```bash
# Create volume for n8n data persistence
sudo docker volume create n8n_data

# Run n8n
sudo docker run -d --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

### 3. Set up ngrok for HTTPS on the EC2 Instance

```bash
# Install ngrok
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
sudo tar -xvzf ngrok-v3-stable-linux-amd64.tgz -C /usr/local/bin

# Add authtoken
ngrok config add-authtoken YOUR_TOKEN_HERE

# Install screen
sudo apt-get update
sudo apt-get install screen

# Run ngrok in detached screen session
screen -S ngrok
ngrok http 5678
# Press Ctrl+A then D to detach
```

### 4. Update n8n with ngrok URL

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

### 5. Configure Google OAuth

Set up Google Cloud Console credentials and enable Gmail API. Add your ngrok URL as an authorized redirect URI.

### 6. Import Workflow

Download `workflow.json` and import it into your n8n instance.

Helpful Videos: 
1. AWS instance and n8n setup: https://www.youtube.com/watch?v=GnckYEpY978
2. Configuring Google Cloud credentials: https://www.youtube.com/watch?v=FBGtpWMTppw

## How It Works

**Stage 1 - First Follow-up:**
1. Checks emails labeled 'Initial Outreach'
2. Filters out emails that received a response
3. Identifies emails where 3 business days have passed (excluding Fridays/weekends)
4. Moves them to 'Follow-up 1 Await' label
5. Sends notification email with approve/diapprove button to ask if label should be changed from 'Follow-up 1 Await' to 'Follow-up 1 Sent'
6. If approved, checks if email thread has 2 sent emails to ensure follow-up was sent indeed, progresses with label change. 

**Stage 2 - Final Follow-up:**
1. Checks emails labeled 'F-1 Sent'
2. Filters out emails with responses
3. Identifies emails where 5 business days have passed (excluding Fridays/weekends)
4. Moves them to 'Final Follow-up Await' label
5. Sends notification email with approve/diapprove button to ask if label should be changed from 'Final Follow-up Await' to 'Final Follow-up Sent'
6. If approved, checks if email thread has 3 sent emails to ensure follow-up was sent indeed, progresses with label change. 

## Usage

1. Send initial outreach email
2. Create follow-up draft as a reply in the same thread
3. Label the sent email as 'Initial Outreach'
4. System automatically tracks and notifies when follow-up is due
5. Send the draft when notified
6. Approve label change to 'Follow-up 1 Sent'
7. System tracks for final follow-up

## Why This Approach?

- **Drafts over auto-send:** Maintains personal voice and control 
- **Gmail threading:** Keeps conversations organized
- **Business day logic:** Respects professional communication norms
- **Response filtering:** Automatically excludes emails that got replies

## Cost

- AWS: Free tier (750 hours during 1st year)
- ngrok: Free tier (note: URL changes on restart of AWS instance)
- n8n: Open source
