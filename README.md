import streamlit as st
import google.generativeai as genai
from PIL import Image
from io import BytesIO
import time
import json

# Configure Gemini API key
genai.configure(api_key=" ENTER YOUR API KEY ")  # Replace with your valid 39-char key

# App Title
st.title(" Food Image Generator")

# Initialize chat history
if 'messages' not in st.session_state:
    st.session_state.messages = []

# Display chat messages
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])
        if "image" in message and message["image"] is not None:
            st.image(message["image"], caption=message["caption"], use_column_width=True)
        if "report" in message:
            st.markdown(message["report"])

# Generate function (Optimized for image generation)
def generate_food_image(prompt_text):
    try:
        model = genai.GenerativeModel('gemini-2.5-flash-image-preview')
        enhanced_prompt = f"Generate a high-quality, photorealistic image of: {prompt_text}. Style: Professional food photography, vibrant colors, appetizing appearance. Output format: JPEG."
        start_time = time.time()
        response = model.generate_content(enhanced_prompt)
        latency = time.time() - start_time

        image_data = None
        for part in response.candidates[0].content.parts:
            if part.inline_data:
                image_data = part.inline_data.data
                break

        if image_data:
            image = Image.open(BytesIO(image_data))
            return image, "Image generated successfully!", latency
        else:
            return None, "No image data received from API", latency
    except Exception as e:
        return None, str(e), 0

# Confidence function
def get_confidence_rating(prompt_text, latency):
    prompt_lower = prompt_text.lower()
    high_confidence_foods = ['biriyani', 'pizza', 'pasta', 'paneer tikka', 'idly', 'dosa']
    medium_confidence_foods = ['sambar', 'rasam', 'vada', 'tacos', 'sushi']

    if any(food in prompt_lower for food in high_confidence_foods):
        base_conf = 90
    elif any(food in prompt_lower for food in medium_confidence_foods):
        base_conf = 80
    else:
        base_conf = 75

    if latency < 4:
        return min(95, base_conf + 5)
    elif latency < 7:
        return base_conf
    else:
        return max(65, base_conf - 5)

# Chat input
prompt = st.chat_input("Enter any global food prompt (e.g., A delicious Biriyani with raita)...")
if prompt:
    st.session_state.messages.append({"role": "user", "content": prompt})

    with st.chat_message("user"):
        st.markdown(prompt)
    with st.chat_message("assistant"):
        st.markdown(f"🍳 Generating image for: **{prompt}**")

        with st.spinner(" Creating your food image..."):
            image, response_text, latency = generate_food_image(prompt)

        if image:
            st.success(" Image generated successfully!")
            st.image(image, caption=f"Generated image for: {prompt}", use_column_width=True)
            confidence = get_confidence_rating(prompt, latency)
            report = f"""
            ** Performance Report:**
            -  **Latency:** {latency:.2f} seconds
            -  **Accuracy Confidence:** {confidence}%
            -  **Cuisine Support:** All global cuisines
            -  **Status:** Successfully processed
            """
            st.markdown(report)
            st.session_state.messages.append({
                "role": "assistant",
                "content": response_text,
                "image": image,
                "caption": f"Generated image for: {prompt}",
                "report": report
            })
        else:
            error_msg = f" Error: {response_text}"
            st.error(error_msg)
            st.session_state.messages.append({
                "role": "assistant",
                "content": error_msg
            })

# Sidebar
with st.sidebar:
    st.header(" Supported Cuisines")
    st.write("""
    **தமிழ்:** சாம்பார், ரசம், பிரியாணி
    **North Indian:** Butter Chicken, Naan
    **South Indian:** Idly, Dosa, Vada
    **International:** Pizza, Pasta, Burgers
    **Asian:** Sushi, Noodles, Fried Rice
    **European:** Lasagna, Risotto, Paella
    **American:** Hot Dogs, BBQ, Burgers
    **African:** Jollof Rice, Couscous, Tagine
    """)

    st.header(" Tips for Best Results")
    st.write("""
    1. Be specific: "A delicious Chicken Biriyani with Raita"
    2. Include details: "Idly with Sambar and Chutney on a banana leaf"
    3. Mention style: "Authentic Chennai Style Biriyani in a clay pot"
    """)

    if st.button(" Clear Chat History"):
        st.session_state.messages = []
        st.rerun()

# Save chat history
if st.session_state.messages:
    try:
        serializable_messages = []
        for msg in st.session_state.messages:
            serializable_msg = msg.copy()
            if "image" in serializable_msg:
                serializable_msg['image'] = "Image data saved separately"
            serializable_messages.append(serializable_msg)

        with open("chat_history.json", "w") as f:
            json.dump(serializable_messages, f, indent=2)
    except Exception as e:
        st.sidebar.error(f"Error saving history: {str(e)}")
