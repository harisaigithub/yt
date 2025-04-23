# 🎬 YouTube Summarizer - GCP Deployment Guide

This guide helps you deploy the **YouTube Summarizer** app to Google Cloud Run.

---

## 🚀 Prerequisites
1. Open your L4G Process Document. <a href="https://docs.google.com/document/u/0/d/14Qc921t8DxpqdHzHinwovwWJtR9e70WD_ktAe8qX3fs/mobilebasic" target="_blank">bit.ly/L4GWorkshop</a>
2. Navigate through this Process Document to complete the prerequisites ([https://tinyurl.com/l4gprerequisites](https://vvitguntur-my.sharepoint.com/:b:/g/personal/22bq5a4213_vvit_net/EWf5yXQqheJMq776JOsm8HoBjx1YNyhSstH5PtT3vFAhAw?e=5VFS5M))
3. To Claim the credits(only after completion of registration) DOC. ([https://tinyurl.com/l4gcredits](https://vvitguntur-my.sharepoint.com/:b:/g/personal/22bq5a4213_vvit_net/EU8_-DFd1VJAm_eFNCnQTxsBDWQasMScNYf_lg7S6RMnLQ?e=6fsA6O))
4. You must have access to a Google Cloud account.
5. You must have `gcloud` CLI installed and authenticated.
6. Enable billing on your GCP project. 

---

## 📌 Steps to Deploy

### 1. Select or Create a GCP Project
- Go to [Google Cloud Console](https://console.cloud.google.com/).
- Select an existing project or click on **"New Project"** to create one.
  ![image](https://github.com/user-attachments/assets/336e918d-dad3-44df-9ded-199a6340301e)
- Once selected, click the terminal icon (cloud shell) on the top-right to open the console.
  ![image](https://github.com/user-attachments/assets/4331cf1d-d86e-4a86-b8e0-3802e649d73d)
- Click **Continue** and **Authorize** if prompted.
  ![image](https://github.com/user-attachments/assets/2f63f6c2-55c6-445d-8301-cc2bb4eb1457)
- The shell will display your active project ID.
  ![image](https://github.com/user-attachments/assets/4c4f75a4-c01e-49c0-8453-2af623a2fce4)


# ⚠️ If you don't see a project ID, raise your hands we will try to resolve it :] ..

---

### 2. Create the Script

Run this command in the terminal:

```bash
nano youtubesummarize.sh
```

Paste the following content into the opened editor:

```bash

#!/bin/bash

# Set this manually or export it before running the script
# Example: export PROJECT_ID="your-project-id"
PROJECT_ID=$(gcloud config get-value project)

# Exit if project is not set
if [ -z "$PROJECT_ID" ]; then
  echo "❌ GCP Project ID not set. Please run: gcloud config set project [PROJECT_ID]"
  exit 1
fi

APP_NAME="youtube-summarizer"
REGION="us-central1"
SERVICE_NAME="youtube-summarizer"

# Step 1: Create working directory
mkdir -p $APP_NAME/templates
cd $APP_NAME

# Step 2: Create index.html
cat <<'EOF' > templates/index.html
<!DOCTYPE html>
<html>
 <head>
   <title>YouTube Summarizer</title>
   <style>
     body {
       font-family: sans-serif;
       display: flex;
       justify-content: center;
       align-items: center;
       min-height: 100vh;
       background-color: #f4f4f4;
     }
     .container {
       background-color: white;
       padding: 30px;
       border-radius: 8px;
       box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
       text-align: center;
     }
     h2 {
       margin-bottom: 20px;
     }
     input[type="text"], textarea, select {
       width: 100%;
       padding: 10px;
       margin-bottom: 15px;
       border: 1px solid #ccc;
       border-radius: 4px;
     }
     button {
       background-color: #4CAF50;
       color: white;
       padding: 12px 20px;
       border: none;
       border-radius: 4px;
       cursor: pointer;
     }
   </style>
 </head>
 <body>
   <div class="container">
     <h2>YouTube Summarizer</h2>
     <form action="/summarize" method="post" target="_blank">
       <input type="text" name="youtube_link" placeholder="Enter YouTube Link" required>
       <select name="model">
         <option value="gemini-2.0-flash-001">Gemini 2.0 Flash</option>
       </select>
       <textarea name="additional_prompt" placeholder="Optional prompt (e.g., 'Explain like I’m five')"></textarea>
       <button type="submit">Summarize</button>
     </form>
   </div>
 </body>
</html>
EOF

# Step 3: Create requirements.txt
cat <<EOF > requirements.txt
Flask==2.3.3
requests==2.31.0
debugpy
google-genai==1.2.0
EOF

# Step 4: Create app.py
cat <<EOF > app.py
import os
from flask import Flask, render_template, request, redirect
from google import genai
from google.genai import types

app = Flask(__name__)
PROJECT_ID = "$PROJECT_ID"

client = genai.Client(
   vertexai=True,
   project=PROJECT_ID,
   location="us-central1",
)

@app.route('/', methods=['GET'])
def index():
   return render_template('index.html')

def generate(youtube_link, model, additional_prompt):
   youtube_video = types.Part.from_uri(
       file_uri=youtube_link,
       mime_type="video/*",
   )

   if not additional_prompt:
       additional_prompt = " "

   contents = [
       youtube_video,
       types.Part.from_text(text="Provide a summary of the video."),
       additional_prompt,
   ]

   generate_content_config = types.GenerateContentConfig(
       temperature=1,
       top_p=0.95,
       max_output_tokens=8192,
       response_modalities=["TEXT"],
   )

   return client.models.generate_content(
       model=model,
       contents=contents,
       config=generate_content_config,
   ).text

@app.route('/summarize', methods=['GET', 'POST'])
def summarize():
   if request.method == 'POST':
       youtube_link = request.form['youtube_link']
       model = request.form['model']
       additional_prompt = request.form['additional_prompt']
       try:
           summary = generate(youtube_link, model, additional_prompt)
           return summary
       except Exception as e:
           return f"❌ Error: {str(e)}"
   else:
       return redirect('/')

if __name__ == '__main__':
   server_port = os.environ.get('PORT', '8080')
   app.run(debug=False, port=server_port, host='0.0.0.0')
EOF

# Step 5: Create Dockerfile
cat <<EOF > Dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
EOF

# Step 6: Enable necessary APIs
echo "⏳ Enabling required Google Cloud APIs..."
gcloud services enable aiplatform.googleapis.com \
                       run.googleapis.com \
                       cloudbuild.googleapis.com \
                       cloudresourcemanager.googleapis.com

# Step 7: (Optional) Local Testing Setup
echo "🔧 Creating virtual environment for local testing..."
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate

# Step 8: Deploy to Cloud Run
echo "🚀 Deploying to Cloud Run..."
gcloud run deploy $SERVICE_NAME \
  --source . \
  --region=$REGION \
  --allow-unauthenticated \
  --project=$PROJECT_ID

echo "✅ Deployment complete!"

```

Save and exit:
- Press `Ctrl + O`, then `Enter` to save.
- Press `Ctrl + X` to exit the editor.

---

### 3. Make the Script Executable

```bash
chmod +x youtubesummarize.sh
```

---

### 4. Run the Script

```bash
./youtubesummarize.sh
```

If everything goes fine, you will see:

```
✅ Deployment complete!
```

Along with a URL like:

> `https://youtube-summarizer-xyz123.a.run.app/`
---

## ✅ Final Check

Open the generated URL in a browser. You should see the **YouTube Summarizer** app interface.
![image](https://github.com/user-attachments/assets/ddef9af5-0ab7-4131-b0c0-dad2814ffc11)


---

## 🎉 Congrats!

You’ve successfully deployed your app to Google Cloud Run.

> **Made with 💚 by RAVI, HARI, RUTHWIK** @team-L4G
