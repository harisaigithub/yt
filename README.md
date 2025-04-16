# yt
#DEPLOY

```
nano yt.sh
```


```
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
```
chmod +x yt.sh
```
```
./yt.sh
```

