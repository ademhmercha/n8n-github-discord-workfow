# Setup Guide

## Prerequisites

- Node.js 18+ (LTS recommended)
- A Discord server where you have "Manage Webhooks" permission
- A GitHub repository you own or administer

---

## Step 1: Install n8n

### Option A: Local Installation (Recommended for Development)

```bash
npm install n8n -g
```

### Option B: Docker Installation

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

### Option C: n8n Cloud

Create an account at [n8n.cloud](https://n8n.cloud) and follow their onboarding flow.

---

## Step 2: Launch n8n

```bash
n8n start
```

Navigate to [http://localhost:5678](http://localhost:5678) in your browser. Create your user account on first launch.

---

## Step 3: Import the Workflow

1. Open n8n in your browser
2. Click **Workflows** in the left sidebar
3. Click the **Import** button in the top-right corner
4. Select the file `workflow/github-discord-workflow.json` from this repository
5. The workflow will appear in the editor with all four pre-configured nodes

---

## Step 4: Configure the JavaScript Code Node

Open the **Build Discord Embed** node and paste the following JavaScript code:

```javascript
const event = $input.first().json.headers['x-github-event'];
const payload = $input.first().json.body;

let embeds = [];

if (event === 'push') {
  const repo = payload.repository.full_name;
  const branch = payload.ref.replace('refs/heads/', '');
  const commits = payload.commits || [];
  const pusher = payload.pusher.name;
  const commitCount = commits.length;

  const description = commits.slice(0, 5).map(c =>
    `[\`${c.id.substring(0, 7)}\`](${c.url}) ${c.message.split('\n')[0]}`
  ).join('\n');

  const totalCommits = commitCount > 5
    ? `\n\n... and ${commitCount - 5} more commits`
    : '';

  embeds.push({
    title: `🚀 [${repo}] New Push to ${branch}`,
    url: `https://github.com/${repo}/commits/${branch}`,
    color: 0x2ea043,
    description: description + totalCommits,
    fields: [
      {
        name: '📦 Repository',
        value: `[${repo}](https://github.com/${repo})`,
        inline: true
      },
      {
        name: '🌿 Branch',
        value: `\`${branch}\``,
        inline: true
      },
      {
        name: '📝 Commits',
        value: `${commitCount}`,
        inline: true
      },
      {
        name: '👤 Author',
        value: pusher,
        inline: true
      }
    ],
    timestamp: new Date().toISOString(),
    footer: {
      text: 'GitHub Discord Notification Bot',
      icon_url: 'https://github.githubassets.com/favicons/favicon.svg'
    }
  });
}

if (event === 'pull_request') {
  const pr = payload.pull_request;
  const repo = payload.repository.full_name;
  const action = payload.action;
  const state = pr.merged ? 'merged' : pr.state;

  let color;
  let actionEmoji;

  switch (action) {
    case 'opened':
      color = 0x2ea043;
      actionEmoji = '📗';
      break;
    case 'closed':
      color = pr.merged ? 0x8250df : 0xcf222e;
      actionEmoji = pr.merged ? '🎉' : '📕';
      break;
    case 'reopened':
      color = 0x2ea043;
      actionEmoji = '📗';
      break;
    case 'synchronize':
      color = 0xe3b341;
      actionEmoji = '🔄';
      break;
    default:
      color = 0x6e7681;
      actionEmoji = 'ℹ️';
  }

  embeds.push({
    title: `${actionEmoji} [${repo}] Pull Request ${state === 'merged' ? 'Merged' : action.charAt(0).toUpperCase() + action.slice(1)}`,
    url: pr.html_url,
    color: color,
    description: `### [${pr.title}](${pr.html_url})`,
    fields: [
      {
        name: '📦 Repository',
        value: `[${repo}](https://github.com/${repo})`,
        inline: true
      },
      {
        name: '🔀 PR Number',
        value: `#${pr.number}`,
        inline: true
      },
      {
        name: '📋 Status',
        value: pr.merged ? 'Merged' : pr.state,
        inline: true
      },
      {
        name: '👤 Author',
        value: pr.user.login,
        inline: true
      },
      {
        name: '🌿 Base',
        value: `\`${pr.base.ref}\``,
        inline: true
      },
      {
        name: '🔀 Head',
        value: `\`${pr.head.ref}\``,
        inline: true
      }
    ],
    timestamp: new Date().toISOString(),
    footer: {
      text: 'GitHub Discord Notification Bot',
      icon_url: 'https://github.githubassets.com/favicons/favicon.svg'
    }
  });
}

return { embeds };
```

Click **Save** after pasting.

---

## Step 5: Create a Discord Webhook

1. Open your Discord server
2. Go to **Server Settings** → **Integrations** → **Webhooks**
3. Click **Create Webhook**
4. Give it a name (e.g., "GitHub Notifications")
5. Select the channel where notifications should appear
6. Click **Copy Webhook URL** and save it

---

## Step 6: Configure Environment Variables

1. In n8n, go to **Settings** → **Environment Variables**
2. Add a variable:
   - **Name**: `DISCORD_WEBHOOK_URL`
   - **Value**: `https://discord.com/api/webhooks/your-webhook-id/your-webhook-token`

> **Security Note**: Never commit the webhook URL to version control. Always use environment variables or n8n credentials.

---

## Step 7: Expose n8n to the Internet

GitHub must be able to reach your n8n instance. Choose one of the following options:

### Option A: ngrok (Quickest for Testing)

```bash
ngrok http 5678
```

Copy the HTTPS URL (e.g., `https://abc123.ngrok.io`) and note the `/webhook/github-webhook` path.

### Option B: Deploy to a VPS

Use a cloud provider (DigitalOcean, AWS, Hetzner) and follow the Docker deployment guide.

### Option C: n8n Cloud

If using n8n Cloud, your instance already has a public URL.

---

## Step 8: Configure GitHub Webhook

1. Go to your GitHub repository
2. Click **Settings** → **Webhooks** → **Add webhook**
3. Configure the following:

| Field | Value |
|-------|-------|
| **Payload URL** | `https://your-n8n-instance.com/webhook/github-webhook` |
| **Content type** | `application/json` |
| **Secret** | Leave blank (optional but recommended) |
| **SSL verification** | Enable (or disable for testing with ngrok) |
| **Which events** | Select **Let me select individual events** |
| - Push | ✅ |
| - Pull requests | ✅ |
| **Active** | ✅ |

4. Click **Add webhook**

---

## Step 9: Activate the Workflow

1. In n8n, click the **Inactive** toggle at the top-right of the workflow editor
2. The toggle should turn green and display **Active**
3. Your workflow is now live and listening for GitHub events

---

## Step 10: Test the Integration

### Test Push Events

Make a commit and push to any branch in your repository:

```bash
git commit --allow-empty -m "test: verify discord notification"
git push origin main
```

### Test Pull Request Events

1. Create a new branch: `git checkout -b test-notifications`
2. Make a small change, commit, and push
3. Open a pull request on GitHub
4. Close or merge the pull request

Check your Discord channel — you should see rich embed notifications for each event.

---

## Troubleshooting

### Workflow not triggering

- Verify the webhook URL is publicly accessible
- Check **GitHub → Settings → Webhooks → Recent Deliveries** for response codes
- Ensure the workflow is in **Active** state

### Discord embed not appearing

- Verify `DISCORD_WEBHOOK_URL` environment variable is set correctly
- Test the webhook URL with curl:
  ```bash
  curl -X POST -H "Content-Type: application/json" \
    -d '{"content":"test"}' \
    $DISCORD_WEBHOOK_URL
  ```

### Events being ignored

- The IF node only allows `push` and `pull_request` events by design
- Check the `x-github-event` header in GitHub's delivery log
- Extend the IF node conditions if you want to support additional events

---

## Next Steps

- Read [architecture.md](./architecture.md) for a deep dive into the workflow design
- Explore the `examples/` directory for sample payloads
- Check the [main README](../README.md) for future improvement ideas
