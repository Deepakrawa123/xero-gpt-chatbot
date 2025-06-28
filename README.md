# xero-gpt-chatbot

import os
import json
import requests
from flask import Flask, redirect, request, render_template_string
from requests_oauthlib import OAuth2Session
from openai import OpenAI
from dotenv import load_dotenv

# Load .env variables (for local dev)
load_dotenv()

# === Flask App ===
app = Flask(__name__)

# === Configuration ===
CLIENT_ID = os.getenv("CLIENT_ID")
CLIENT_SECRET = os.getenv("CLIENT_SECRET")
REDIRECT_URI = os.getenv("REDIRECT_URI")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

AUTH_BASE_URL = 'https://login.xero.com/identity/connect/authorize'
TOKEN_URL = 'https://identity.xero.com/connect/token'
API_BASE_URL = 'https://api.xero.com/api.xro/2.0'

# === Root Route ===
@app.route('/')
def login():
    xero = OAuth2Session(CLIENT_ID, redirect_uri=REDIRECT_URI, scope=['accounting.reports.read'])
    auth_url, _ = xero.authorization_url(AUTH_BASE_URL)
    return redirect(auth_url)

# === Callback Route ===
@app.route('/callback')
def callback():
    try:
        xero = OAuth2Session(CLIENT_ID, redirect_uri=REDIRECT_URI)
        token = xero.fetch_token(
            TOKEN_URL,
            client_secret=CLIENT_SECRET,
            authorization_response=request.url,
        )

        access_token = token['access_token']
        headers = {
            "Authorization": f"Bearer {access_token}",
            "Accept": "application/json"
        }

        # 🔑 Get Tenant ID
        connections = requests.get("https://api.xero.com/connections", headers=headers).json()
        if not connections or "tenantId" not in connections[0]:
            return "❌ Error: Unable to retrieve tenant ID."

        tenant_id = connections[0]["tenantId"]
        headers["Xero-tenant-id"] = tenant_id

        # 📊 Get Financial Reports
        pl_data = requests.get(f"{API_BASE_URL}/Reports/ProfitAndLoss", headers=headers).json()
        bs_data = requests.get(f"{API_BASE_URL}/Reports/BalanceSheet", headers=headers).json()

        # 🧠 OpenAI Chat (GPT-4o-mini)
        client = OpenAI(api_key=OPENAI_API_KEY)
        prompt = f"""
        Analyze the following Xero financial reports.

        Profit and Loss Report:
        {json.dumps(pl_data)}

        Balance Sheet Report:
        {json.dumps(bs_data)}

        Provide:
        - Summary
        - Key trends
        - Risks or anomalies
        - Strategic recommendations
        """

        chat = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a financial analyst."},
                {"role": "user", "content": prompt}
            ]
        )

        analysis = chat.choices[0].message.content

        return render_template_string("""
            <h1>📊 GPT Financial Analysis</h1>
            <pre style="white-space: pre-wrap;">{{ analysis }}</pre>
        """, analysis=analysis)

    except Exception as e:
        import traceback
        tb = traceback.format_exc()
        return f"<h2>❌ Internal Server Error</h2><pre>{tb}</pre>"

# === Run App Locally ===
if __name__ == '__main__':
    app.run(debug=True, port=5000)
