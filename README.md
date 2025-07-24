# Deploy your agent on Agentverse via Render

[Render](https://render.com/) is a cloud platform that lets you deploy Python web services and agents with ease. It supports Docker and is perfect for deploying uAgents that need to be accessible on the public internet, with health checks and mailbox support for Agentverse.

---

## Step-by-Step: Deploy Your Agent

### 1. Sign up for Render
Go to [render.com](https://render.com/) and create a free account.

### 2. Prepare your repo
Make sure your repo contains:
- `agent.py` (your agent code)
- `Dockerfile`
- `requirements.txt`
- `README.md` (optional, but recommended)

### 3. Go to the Render Dashboard and create a new service
Click the **+ New** button in the top right.

![Render dashboard new service](https://render.com/docs-assets/7dc4f6883d3ea0c4791a40442153805bb0f8c8f3edf647cf599e601e016117cb/new-dropdown.webp)

Choose **Web Service**.

### 4. Link your repo
After you select Web Service, connect your GitHub (or GitLab/Bitbucket) account to Render.

![Link your repo](https://render.com/docs-assets/204d0deba469fef755a7eac471eb09b52ebeaa12d53b3b426dee1ae969cfd004/git-connect.webp)

After you connect, the form shows a list of all the repos you have access to. Select the repo that contains your agent and click **Connect**.

The rest of the creation form appears.

### 5. Configure and deploy
- Set the **branch** (e.g., `main`)
- Set the **port** to `8000` (or your FastAPI port)
- Make sure your Dockerfile is detected
- Add any required environment variables (e.g., API keys)
- Click **Create Web Service** and wait for the build and deploy to finish.

### 6. Monitor your deploy
Render automatically opens a log explorer that shows your deploy's progress:

![Render deploy logs](https://render.com/docs-assets/6aa330511f718bd38326c7f6c0fd8697807e4c3ae18257449d156ae353ca1a09/first-deploy-logs.webp)

If the deploy completes successfully, the deploy's status updates to **Live** and you'll see log lines like:
```
==> Deploying...
==> Running 'python3 agent.py'
==> Your service is live 🎉
```
If the deploy fails, the deploy's status updates to **Failed**. Review the log feed to help identify the issue.

### 7. Test your agent locally with Docker

**requirements.txt**
```
uagents
requests
openai
fastapi
```

**Dockerfile**
```
FROM python:3.12
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python3", "agent.py"]
```

**Build and run locally:**
```sh
# Build the Docker image
docker build -t testing .
# Run the container (exposes port 8000)
docker run -it testing
```

### 8. Test the health endpoint
```sh
curl http://localhost:8000/ping
# Should return: {"status": "agent is running"}
```

### 9. After deployment, test your health endpoint on Render
```sh
curl https://your-agent-service.onrender.com/ping
# Should return: {"status": "agent is running"}
```

### 10. Your agent is now live
You can now register your agent on Agentverse with the public Render URL.

---

## Full Example:

**agent.py**
```python
from datetime import datetime
from uuid import uuid4
from threading import Thread

from fastapi import FastAPI
import uvicorn
from openai import OpenAI
from uagents import Context, Protocol, Agent
from uagents_core.contrib.protocols.chat import (
    ChatAcknowledgement,
    ChatMessage,
    TextContent,
    chat_protocol_spec,
)

app = FastAPI()

@app.get("/ping")
def ping():
    return {"status": "agent is running"}

client = OpenAI(
    base_url='https://api.asi1.ai/v1',
    api_key='YOUR_ASI_ONE_API_KEY',
)

agent = Agent(
    name="general-ai-agent",
    seed="general-ai-agent-seed",
    port=8002,
    mailbox=True,
    publish_agent_details=True,
)

protocol = Protocol(spec=chat_protocol_spec)

@protocol.on_message(ChatMessage)
async def handle_message(ctx: Context, sender: str, msg: ChatMessage):
    await ctx.send(sender, ChatAcknowledgement(timestamp=datetime.now(), acknowledged_msg_id=msg.msg_id))
    text = ''.join([item.text for item in msg.content if isinstance(item, TextContent)])
    try:
        r = client.chat.completions.create(
            model="asi1-mini",
            messages=[
                {"role": "system", "content": "You are a helpful, knowledgeable AI assistant. Answer the user's questions to the best of your ability."},
                {"role": "user", "content": text},
            ],
            max_tokens=2048,
        )
        response = str(r.choices[0].message.content)
    except Exception:
        ctx.logger.exception('Error querying model')
        response = "I am afraid something went wrong and I am unable to answer your question at the moment"
    await ctx.send(sender, ChatMessage(
        timestamp=datetime.utcnow(),
        msg_id=uuid4(),
        content=[TextContent(type="text", text=response)],
    ))

@protocol.on_message(ChatAcknowledgement)
async def handle_ack(ctx: Context, sender: str, msg: ChatAcknowledgement):
    pass

agent.include(protocol, publish_manifest=True)

def run_agent():
    agent.run()

if __name__ == "__main__":
    Thread(target=run_agent).start()
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## README example for Agentverse

```
![tag:restaurant](https://img.shields.io/badge/restaurant-3D8BD3)

![tag:innovationlab](https://img.shields.io/badge/innovationlab-3D8BD3)

This agent helps you to book a table at the Mexican Gourmet Restaurant by taking your name and phone number.
```

**Input Data Model**
```
class ReservationRequest(Model):
    name: str
    phone_number: str
```

**Output Data Model**
```
class ReservationResponse(Model):
    message: str
```
```
```
## Health Check

```sh
curl https://your-agent-service.onrender.com/ping
# {"status": "agent is running"}
```
```
