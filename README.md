# AI Storyteller Trip Planner - Hackathon Prototype Guide

This guide provides a complete, step-by-step walkthrough for building the "AI Storyteller Trip Planner" prototype. It covers everything from cloud setup to final deployment and is designed to help you create a feature-rich, impressive application for the hackathon.

-----

## Milestone 1: The Complete Cloud Foundation

**Objective:** To meticulously create and configure every cloud service you will need from scratch.

### 1\. Create Google Cloud Project

  - Go to the [GCP Console](https://console.cloud.google.com/).
  - In the top bar, click the project dropdown and select **NEW PROJECT**.
  - Enter a **Project name**: `genai-hackathon-winner`. Click **CREATE**.
  - From the main navigation menu (☰), go to **Billing** and ensure your new project is linked to your billing account to activate the free credits.

### 2\. Enable All Necessary APIs

  - In the main navigation menu, go to **APIs & Services -\> Library**.
  - Search for and **ENABLE** each of the following APIs one by one:
      - Vertex AI API
      - Cloud Functions API
      - Cloud Build API
      - Cloud Run Admin API
      - Cloud Firestore API
      - Identity and Access Management (IAM) API
      - BigQuery API
      - Google Maps Platform

### 3\. Create Your Maps API Key

  - In the **APIs & Services** section, go to the **Credentials** tab.
  - Click **+ CREATE CREDENTIALS** and select **API key**.
  - A key will be generated. Click the copy icon and paste this key into a secure text file. You will need it for your frontend.

### 4\. Set Up BigQuery for Analytics

  - In the GCP Console navigation menu, find the **ANALYTICS** section and click on **BigQuery**.
  - In the **Explorer** panel on the left, find and click on your `genai-hackathon-winner` project.
  - Click the three vertical dots next to your project name and select **Create dataset**.
      - **Dataset ID:** `trip_analytics`
      - **Location type:** Leave as default or choose a location like `US (us-central1)`.
      - Click **CREATE DATASET**.
  - In the Explorer panel, find your new `trip_analytics` dataset. Click the three dots next to it and select **Create table**.
      - **Table name:** `generated_trips`
      - In the **Schema** section, toggle the **Edit as text** option.
      - Paste the following JSON exactly:
        ```json
        [
          {"name": "trip_id", "type": "STRING", "mode": "REQUIRED"},
          {"name": "prompt", "type": "STRING", "mode": "NULLABLE"},
          {"name": "title", "type": "STRING", "mode": "NULLABLE"},
          {"name": "language", "type": "STRING", "mode": "NULLABLE"},
          {"name": "generated_at", "type": "TIMESTAMP", "mode": "REQUIRED"}
        ]
        ```
      - Click **CREATE TABLE**.

### 5\. Set Up Firebase Project

  - Go to the [Firebase Console](https://console.firebase.google.com/).
  - Click **Add project** and select your existing `genai-hackathon-winner` GCP project.
  - **Upgrade to the Blaze Plan:** This is mandatory. On your main Firebase project page, find the plan name (e.g., Spark) in the bottom-left menu. Click it to **Upgrade**, and select the **Blaze (Pay as you go)** plan.
  - **Set up Firestore:** Go to **Build -\> Firestore Database**. Click **Create database**. Start in **Production mode** and choose a location.
  - **Set up Authentication:** Go to **Build -\> Authentication**. Click **Get Started**. In the **Sign-in method** tab, select **Google**, enable it, provide a project support email, and click **Save**.

-----

## Milestone 2: The Advanced AI Backend

**Objective:** To initialize the project locally and deploy two powerful Firebase Functions for generation and real-time adjustments.

### 1\. Initialize Local Project Environment

  - Open your computer's terminal or command prompt.
  - Install the Firebase tools globally:
    ```bash
    npm install -g firebase-tools
    ```
  - Log in to your Google account:
    ```bash
    firebase login
    ```
  - Create your project folder and navigate into it:
    ```bash
    mkdir trip-planner-pro && cd trip-planner-pro
    ```
  - Initialize your Firebase project here:
    ```bash
    firebase init
    ```
  - Follow these specific prompts:
      - **Which Firebase features do you want to set up?** Use arrow keys and spacebar to select **Firestore**, **Functions**, and **Hosting**. Press Enter.
      - **Please select an option:** Choose **Use an existing project**.
      - **Select a default Firebase project:** Choose `genai-hackathon-winner`.
      - **Firestore Rules file?** Press Enter for the default.
      - **Firestore indexes file?** Press Enter for the default.
      - **Language for Functions?** Choose **JavaScript**.
      - **Use ESLint?** Choose **Y**.
      - **Install dependencies with npm now?** Choose **Y**.
      - **Public directory for Hosting?** Type `frontend/build`.
      - **Configure as a single-page app?** Choose **Y**.
      - **Set up automatic builds with GitHub?** Choose **N**.

### 2\. Code the Backend Logic

  - Navigate into the functions directory: `cd functions`

  - Install all necessary backend dependencies:

    ```bash
    npm install @google-cloud/vertexai @google-cloud/bigquery uuid
    ```

  - Create the file `functions/index.js` and add the following code:

    ````javascript
    const { onCall } = require("firebase-functions/v2/https");
    const { VertexAI } = require('@google-cloud/vertexai');
    const { BigQuery } = require('@google-cloud/bigquery');
    const { v4: uuidv4 } = require('uuid');
    const logger = require("firebase-functions/logger");

    // --- INITIALIZE GOOGLE CLOUD CLIENTS ---
    const vertex_ai = new VertexAI({ project: "genai-hackathon-winner", location: "us-central1" });
    const bigquery = new BigQuery();
    const model = 'gemini-1.5-pro-001';

    // --- FUNCTION 1: GENERATE THE FULL ITINERARY ---
    const GENERATE_PROMPT = `
    You are 'Roamer', an expert AI travel guide. Your goal is to create a complete and captivating travel itinerary.
    You MUST respond in the language specified by "language_code".
    Your entire response MUST be a single, valid JSON object with this exact structure:
    {
      "tripTitle": "...",
      "summary": "...",
      "totalEstimatedCost": "...",
      "suggestedAccommodations": [ { "name": "...", "type": "e.g., Luxury Hotel, Budget Hostel", "reason": "..." } ],
      "suggestedTransport": [ { "type": "e.g., Metro, Auto-rickshaw", "details": "..." } ],
      "itinerary": [
        {
          "day": 1,
          "theme": "...",
          "narrative": "...",
          "activities": [
            { "time": "...", "title": "...", "description": "...", "locationName": "...", "location": { "lat": 12.345, "lng": 12.345 }, "estimatedCost": "..." }
          ]
        }
      ]
    }
    For "totalEstimatedCost", provide a currency and a realistic range (e.g., "₹40,000 - ₹50,000 INR").
    For "suggestedAccommodations", suggest 2-3 relevant options.
    For "suggestedTransport", suggest the best ways to get around the city.
    The 'narrative' must be an engaging story.
    `;

    exports.generateItinerary = onCall(async (request) => {
        const { prompt, language } = request.data;
        if (!prompt) throw new functions.https.HttpsError('invalid-argument', 'Missing prompt.');

        const generativeModel = vertex_ai.getGenerativeModel({ model });
        const fullPrompt = `${GENERATE_PROMPT}\n\nlanguage_code: ${language || 'en'}\nUser Request: ${prompt}`;

        try {
            const resp = await generativeModel.generateContent(fullPrompt);
            const content = resp.response.candidates[0].content.parts[0].text;
            const jsonString = content.replace(/```json/g, "").replace(/```/g, "").trim();
            const itineraryData = JSON.parse(jsonString);

            bigquery.dataset('trip_analytics').table('generated_trips').insert([{
                trip_id: uuidv4(), prompt, title: itineraryData.tripTitle, language: language || 'en', generated_at: new Date().toISOString()
            }]).catch(err => logger.error("BigQuery Log Error:", err));

            return { status: "success", data: itineraryData };
        } catch (error) {
            logger.error("Generate Itinerary Error:", error);
            throw new functions.https.HttpsError('internal', 'Failed to generate itinerary.');
        }
    });

    // --- FUNCTION 2: MAKE REAL-TIME ADJUSTMENTS ---
    const ADJUST_PROMPT = `
    You are an expert travel assistant. The user has an existing activity and wants to make a change.
    Analyze the 'current_activity' and the 'adjustment_request'.
    Generate a NEW replacement activity that fulfills the request.
    Your entire response MUST be a single, valid JSON object representing ONLY THE NEW activity in this exact format:
    { "time": "...", "title": "...", "description": "...", "locationName": "...", "location": { "lat": 12.345, "lng": 12.345 }, "estimatedCost": "..." }
    Respond in the language specified by "language_code".
    `;

    exports.adjustItinerary = onCall(async (request) => {
        const { currentActivity, adjustmentRequest, language } = request.data;
        if (!currentActivity || !adjustmentRequest) throw new functions.https.HttpsError('invalid-argument', 'Missing activity or request.');

        const generativeModel = vertex_ai.getGenerativeModel({ model });
        const fullPrompt = `${ADJUST_PROMPT}\n\nlanguage_code: ${language || 'en'}\ncurrent_activity: ${JSON.stringify(currentActivity)}\nadjustment_request: ${adjustmentRequest}`;

        try {
            const resp = await generativeModel.generateContent(fullPrompt);
            const content = resp.response.candidates[0].content.parts[0].text;
            const jsonString = content.replace(/```json/g, "").replace(/```/g, "").trim();
            const adjustedActivity = JSON.parse(jsonString);

            return { status: "success", data: adjustedActivity };
        } catch(error) {
            logger.error("Adjust Itinerary Error:", error);
            throw new functions.https.HttpsError('internal', 'Failed to adjust itinerary.');
        }
    });
    ````

### 3\. Deploy the Backend

  - From the `functions` directory, run:
    ```bash
    firebase deploy --only functions
    ```
  - Wait for the "Deploy complete\!" message.

-----

## Milestone 3: The Feature-Rich Frontend Application

**Objective:** To build the complete React UI that supports every required feature.

### 1\. Create React App and Install Dependencies

  - Navigate back to the project root directory: `cd ..`
  - Create the frontend app:
    ```bash
    npx create-react-app frontend
    ```
  - Navigate into the new frontend directory: `cd frontend`
  - Install all frontend dependencies:
    ```bash
    npm install firebase @vis.gl/react-google-maps
    ```

### 2\. Create Firebase Configuration File

  - In the Firebase Console, go to **Project Settings** (gear icon). Under the **General** tab, scroll to **Your apps**. Click the Web icon (`</>`). Register an app and copy the `firebaseConfig` object.

  - Create a new file at `frontend/src/firebaseConfig.js` and paste the following, inserting your own config object:

    ```javascript
    import { initializeApp } from "firebase/app";
    import { getFunctions, httpsCallable } from "firebase/functions";
    import { getAuth, GoogleAuthProvider, signInWithPopup, onAuthStateChanged, signOut } from "firebase/auth";
    import { getFirestore, collection, addDoc, serverTimestamp } from "firebase/firestore";

    // PASTE YOUR FIREBASE CONFIG OBJECT FROM THE FIREBASE CONSOLE HERE
    const firebaseConfig = {
      apiKey: "AIza...",
      authDomain: "genai-hackathon-winner.firebaseapp.com",
      projectId: "genai-hackathon-winner",
      storageBucket: "genai-hackathon-winner.appspot.com",
      messagingSenderId: "...",
      appId: "..."
    };

    // Initialize Firebase and export the services
    const app = initializeApp(firebaseConfig);
    export const functions = getFunctions(app);
    export const auth = getAuth(app);
    export const db = getFirestore(app);

    // Export auth methods and constants
    export { GoogleAuthProvider, signInWithPopup, onAuthStateChanged, signOut };

    // Export firestore methods
    export { collection, addDoc, serverTimestamp };

    // Create callable references to our backend functions
    export const generateItineraryCallable = httpsCallable(functions, 'generateItinerary');
    export const adjustItineraryCallable = httpsCallable(functions, 'adjustItinerary');
    ```

### 3\. Create the Map View Component

  - Create a new file at `frontend/src/MapView.js` and paste the following code:
    ```javascript
    import React from 'react';
    import { APIProvider, Map, AdvancedMarker } from '@vis.gl/react-google-maps';

    const MapView = ({ itinerary }) => {
      const centerPosition = itinerary?.itinerary[0]?.activities[0]?.location || { lat: 27.1751, lng: 78.0421 }; // Default to Agra
      const allActivities = itinerary?.itinerary.flatMap(day => day.activities) || [];

      return (
        <div className="map-container">
          {/* IMPORTANT: Replace YOUR_GOOGLE_MAPS_API_KEY with the key you saved earlier */}
          <APIProvider apiKey="YOUR_GOOGLE_MAPS_API_KEY">
            <Map
                zoom={12}
                center={centerPosition}
                mapId="TRIP_MAP_ID"
                gestureHandling={'greedy'}
                style={{ width: '100%', height: '100%' }}
            >
              {allActivities.map((activity, index) => (
                <AdvancedMarker key={index} position={activity.location} title={activity.title} />
              ))}
            </Map>
          </APIProvider>
        </div>
      );
    };
    export default MapView;
    ```

### 4\. Create the CSS Stylesheet

  - Open `frontend/src/App.css` and replace its contents with this stylesheet:
    ```css
    /* General Body Styles */
    body {
        margin: 0;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
        'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
        sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        background-color: #f4f7f6;
        color: #333;
    }

    .App {
        text-align: center;
    }

    /* Header */
    .App-header {
        background-color: #ffffff;
        padding: 20px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 15px;
    }

    .header-top {
        width: 100%;
        display: flex;
        justify-content: flex-end;
        align-items: center;
        max-width: 1200px;
    }

    .language-selector {
        margin-right: 20px;
        padding: 8px 12px;
        border-radius: 8px;
        border: 1px solid #ccc;
        background-color: #fff;
    }

    .user-info {
        display: flex;
        align-items: center;
        gap: 10px;
    }

    .user-info img {
        width: 40px;
        height: 40px;
        border-radius: 50%;
    }

    /* Form */
    .prompt-form {
        width: 100%;
        max-width: 800px;
        margin: 30px auto;
        display: flex;
        flex-direction: column;
        gap: 15px;
        padding: 0 20px;
    }

    .prompt-form textarea {
        width: 100%;
        min-height: 100px;
        padding: 15px;
        border-radius: 12px;
        border: 1px solid #ccc;
        font-size: 1rem;
        resize: vertical;
    }

    button {
        padding: 12px 25px;
        font-size: 1rem;
        font-weight: bold;
        border-radius: 12px;
        border: none;
        cursor: pointer;
        background-color: #007bff;
        color: white;
        transition: background-color 0.2s;
    }

    button:hover:not(:disabled) {
        background-color: #0056b3;
    }

    button:disabled {
        background-color: #ccc;
        cursor: not-allowed;
    }

    /* Results Container */
    .results-container {
        display: flex;
        flex-direction: row-reverse;
        gap: 20px;
        padding: 20px;
        max-width: 1400px;
        margin: auto;
    }

    .itinerary-view {
        flex: 3;
        text-align: left;
    }

    .map-container {
        flex: 2;
        min-height: 500px;
        border-radius: 12px;
        overflow: hidden;
        box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }

    /* Cards and Sections */
    .day-card, .recommendation-card {
        background-color: #fff;
        border-radius: 12px;
        padding: 20px;
        margin-bottom: 20px;
        box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }

    .activity {
        border-top: 1px solid #eee;
        padding: 15px 0;
    }

    /* Modal */
    .modal-overlay {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: rgba(0,0,0,0.5);
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .modal-content {
        background: white;
        padding: 30px;
        border-radius: 12px;
        text-align: center;
        width: 90%;
        max-width: 400px;
    }

    .modal-buttons {
        margin-top: 20px;
        display: flex;
        gap: 15px;
        justify-content: center;
    }

    /* Responsive Design */
    @media (max-width: 900px) {
        .results-container {
            flex-direction: column;
        }
        .map-container {
            width: 100%;
            height: 400px;
        }
    }
    ```

### 5\. Create the Main App Component

  - Replace the entire contents of `frontend/src/App.js` with this complete file:
    ```javascript
    import React, { useState, useEffect } from 'react';
    import {
        auth, db, generateItineraryCallable, adjustItineraryCallable,
        GoogleAuthProvider, signInWithPopup, onAuthStateChanged, signOut,
        collection, addDoc, serverTimestamp
    } from './firebaseConfig';
    import MapView from './MapView';
    import './App.css';

    function App() {
        // --- STATE MANAGEMENT ---
        const [user, setUser] = useState(null);
        const [prompt, setPrompt] = useState('');
        const [language, setLanguage] = useState('en');
        const [itinerary, setItinerary] = useState(null);
        const [loading, setLoading] = useState(false);
        const [error, setError] = useState('');
        const [showBookingModal, setShowBookingModal] = useState(false);

        // --- AUTHENTICATION ---
        useEffect(() => {
            const unsubscribe = onAuthStateChanged(auth, currentUser => setUser(currentUser));
            return () => unsubscribe();
        }, []);

        const handleGoogleLogin = async () => await signInWithPopup(auth, new GoogleAuthProvider());
        const handleLogout = async () => await signOut(auth);

        // --- CORE API CALLS ---
        const handleGenerate = async (e) => {
            e.preventDefault();
            if (!prompt) return;
            setLoading(true);
            setError('');
            setItinerary(null);
            try {
                const result = await generateItineraryCallable({ prompt, language });
                if (result.data.status === 'success') {
                    setItinerary(result.data.data);
                } else {
                    throw new Error('API returned an error');
                }
            } catch (err) {
                setError('Failed to generate itinerary. The AI may be overloaded. Please try again.');
                console.error(err);
            } finally {
                setLoading(false);
            }
        };

        const handleAdjust = async (dayIndex, activityIndex) => {
            const adjustmentRequest = window.prompt("How would you like to change this activity? (e.g., 'It's raining, find an indoor alternative nearby')");
            if (!adjustmentRequest) return;

            setLoading(true);
            try {
                const currentActivity = itinerary.itinerary[dayIndex].activities[activityIndex];
                const result = await adjustItineraryCallable({
                    currentActivity,
                    adjustmentRequest,
                    language
                });

                const newItinerary = JSON.parse(JSON.stringify(itinerary)); // Deep copy
                newItinerary.itinerary[dayIndex].activities[activityIndex] = result.data.data;
                setItinerary(newItinerary);

            } catch (err) {
                alert("Failed to make adjustment.");
                console.error(err);
            } finally {
                setLoading(false);
            }
        };

        // --- UTILITY FUNCTIONS ---
        const handleShare = () => {
            if (!itinerary) return;
            navigator.clipboard.writeText(`Check out this trip I planned: ${itinerary.tripTitle}!\n\n${itinerary.summary}`);
            alert("Trip summary copied to clipboard!");
        };

        const handleSaveTrip = async () => {
            if (!user || !itinerary) return;
            try {
              await addDoc(collection(db, "users", user.uid, "trips"), { ...itinerary, savedAt: serverTimestamp() });
              alert("Trip saved to your profile!");
            } catch (err) { console.error("Error saving trip: ", err); alert("Failed to save trip."); }
        };

        return (
            <div className="App">
                <header className="App-header">
                    <div className="header-top">
                        <select className="language-selector" value={language} onChange={e => setLanguage(e.target.value)}>
                            <option value="en">English</option>
                            <option value="hi">हिन्दी (Hindi)</option>
                            <option value="ta">தமிழ் (Tamil)</option>
                            <option value="es">Español (Spanish)</option>
                        </select>
                        {user ? (
                            <div className="user-info">
                                <img src={user.photoURL} alt={user.displayName} />
                                <span>{user.displayName.split(' ')[0]}</span>
                                <button onClick={handleLogout}>Logout</button>
                            </div>
                        ) : <button onClick={handleGoogleLogin}>Sign in with Google</button>}
                    </div>
                    <h1>🚀 AI Storyteller Trip Planner</h1>
                    <p>Describe your dream trip. We'll craft the perfect narrative and visual journey.</p>
                </header>

                <main>
                    <form onSubmit={handleGenerate} className="prompt-form">
                        <textarea value={prompt} onChange={(e) => setPrompt(e.target.value)} placeholder="e.g., A 5-day adventure in Rishikesh for two, focusing on yoga, river rafting, and local cafes. Medium budget." />
                        <button type="submit" disabled={loading}>{loading ? 'Crafting Your Story...' : 'Generate Itinerary'}</button>
                    </form>

                    {error && <p className="error-message">{error}</p>}

                    {itinerary && (
                        <div className="results-container">
                             <div className="itinerary-view">
                                <h2>{itinerary.tripTitle}</h2>
                                <p className="summary">{itinerary.summary}</p>
                                <h3>Total Estimated Cost: {itinerary.totalEstimatedCost}</h3>
                                <div>
                                    {user && <button onClick={handleSaveTrip}>Save Trip</button>}
                                    <button onClick={handleShare}>Share</button>
                                    <button onClick={() => setShowBookingModal(true)}>Book This Trip</button>
                                </div>

                                <div className="recommendation-card">
                                    <h4>Accommodation Suggestions</h4>
                                    <ul>{itinerary.suggestedAccommodations.map((item, i) => <li key={i}><strong>{item.name}</strong> ({item.type}): {item.reason}</li>)}</ul>
                                </div>

                                <div className="recommendation-card">
                                    <h4>Transport Suggestions</h4>
                                    <ul>{itinerary.suggestedTransport.map((item, i) => <li key={i}><strong>{item.type}</strong>: {item.details}</li>)}</ul>
                                </div>

                                {itinerary.itinerary.map((day, dayIndex) => (
                                    <div key={day.day} className="day-card">
                                        <h3>Day {day.day}: {day.theme}</h3>
                                        <p className="narrative">{day.narrative}</p>
                                        {day.activities.map((act, actIndex) => (
                                            <div key={actIndex} className="activity">
                                                <h4>{act.time} - {act.title}</h4>
                                                <p><strong>Location:</strong> {act.locationName}</p>
                                                <p>{act.description}</p>
                                                <p><strong>Est. Cost:</strong> {act.estimatedCost}</p>
                                                <button onClick={() => handleAdjust(dayIndex, actIndex)} disabled={loading}>Adjust</button>
                                            </div>
                                        ))}
                                    </div>
                                ))}
                            </div>
                            <MapView itinerary={itinerary} />
                        </div>
                    )}
                </main>

                {showBookingModal && itinerary && (
                    <div className="modal-overlay">
                        <div className="modal-content">
                            <h2>Confirm Your Trip to {itinerary.tripTitle}</h2>
                            <p>This will proceed to a (mock) payment gateway.</p>
                            <h3>Total Cost: {itinerary.totalEstimatedCost}</h3>
                            <div className="modal-buttons">
                                <button onClick={() => { alert("Booking confirmed! (This is a demo)"); setShowBookingModal(false); }}>Confirm & Pay</button>
                                <button onClick={() => setShowBookingModal(false)} style={{backgroundColor: '#6c757d'}}>Cancel</button>
                            </div>
                        </div>
                    </div>
                )}
            </div>
        );
    }
    export default App;
    ```

-----

## Milestone 4: Final Deployment & The Winning Pitch

**Objective:** To deploy the finished application and prepare key talking points for your presentation.

### 1\. Final Local Test

  - From within the `frontend` directory, run `npm start`.
  - Thoroughly test every feature: Login/logout, generating an itinerary in different languages, clicking "Adjust" on an activity, using the Share button, and opening the mock Booking modal.

### 2\. Build for Production

  - Once satisfied, stop the local server (Ctrl+C).
  - In the `frontend` directory, run the build command:
    ```bash
    npm run build
    ```

### 3\. Deploy to the World

  - Navigate back to the project's root directory: `cd ..`
  - Run the final deploy command:
    ```bash
    firebase deploy --only hosting
    ```
  - Firebase will provide your live URL. This is the link to your finished prototype.

### Your Winning Pitch

Focus your presentation on the interactive "wow" factors you've built:

1.  **The "Smart" Demo:** "Our planner is alive. Watch this." Generate an itinerary. Then, click "Adjust" on an outdoor activity. Say "But what if it's raining? Let's find an indoor museum instead." Type that into the prompt and show the itinerary updating in real-time. This is your most powerful moment.
2.  **The "Inclusive" Demo:** "This is built for everyone." Use the language dropdown to instantly translate the entire rich narrative into Hindi, demonstrating its reach across India.
3.  **The "Complete Vision" Story:** "We've built more than a tool; we've built a platform. From the initial inspirational story, to real-time adjustments, to the final (mock) booking, the user has a seamless experience. And on the backend, we're already gathering trend data in BigQuery to make the next recommendation even smarter."
