import streamlit as st
import re
import pandas as pd
import joblib
from rapidfuzz import fuzz
import tldextract

# --- Load ML models and brand database ---
url_pipeline = joblib.load("svm_url_pipeline.pkl")               # URL classification pipeline
message_pipeline = joblib.load("svm_message_pipeline.pkl")       # Text message classification pipeline
msg_label_encoder = joblib.load("msg_label_encoder.pkl")         # Label encoder for messages
phishing_targets = pd.read_csv("phishing_targets.csv")['label'].str.lower().tolist()  # Known brand names

# --- Extract base domain from URL ---
def get_domain_name(url):
    ext = tldextract.extract(url)
    return ext.domain.lower()

# --- Check if a URL is impersonating a known brand ---
def is_brand_impersonation(url, brand_list, threshold=80):
    domain = get_domain_name(url)
    for brand in brand_list:
        score = fuzz.partial_ratio(brand, domain)
        if score >= threshold:
            return True
    return False

# --- Main scam detection logic ---
def detect_scam(message):
    url_pattern = r'https?://\S+|www\.\S+'
    urls = re.findall(url_pattern, message)
    
    if urls:
        for url in urls:
            if is_brand_impersonation(url, phishing_targets):
                return "Scam (Brand Impersonation)", 1.0
            
            # Use URL classifier model
            pred = url_pipeline.predict([url])[0]
            prob = url_pipeline.predict_proba([url])[0][1]  # Probability of being scam
            if pred == 1:
                return "Scam (Malicious URL)", prob
        
        return "Not Scam", 1.0
    else:
        pred = message_pipeline.predict([message])[0]
        prob = message_pipeline.predict_proba([message])[0][1]  # Probability of spam
        label = msg_label_encoder.inverse_transform([pred])[0]
        return ("Scam", prob) if label == "spam" else ("Not Scam", 1.0 - prob)

# --- Streamlit App UI ---
st.title("📩 ScamSleuth")

input_message = st.text_area("Enter SMS message to check for scam:")

if st.button("Check Scam"):
    if not input_message.strip():
        st.warning("⚠️ Please enter a message.")
    else:
        result, confidence = detect_scam(input_message)
        confidence_pct = round(confidence * 100, 2)
        
        if "not scam" in result.lower():
            st.success(f"👍 {result}")
        else:
            st.error(f"🛑 {result}")
        
        st.info(f"🔎 Confidence: {confidence_pct}%")
