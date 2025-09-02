This will be a complete, step-by-step walkthrough from an empty folder to a fully deployed, feature-rich prototype, with no detail spared.

-----

### **Milestone 1: Cloud Foundation Setup (The Groundwork)**

**Objective:** To meticulously create and configure every cloud service you will need for the entire application.

1.  **Create Google Cloud Project:**

      * Navigate to the [GCP Console](https://console.cloud.google.com/).
      * Click the project dropdown in the top bar and select **NEW PROJECT**.
      * Enter a **Project name**, for example, `hackathon-trip-planner`, and click **CREATE**.

2.  **Link Billing:**

      * Go to the **Billing** section using the main navigation menu.
      * Ensure your new project is selected. If prompted, click **LINK A BILLING ACCOUNT** and associate your account with the free credits.

3.  **Enable All Necessary APIs:**

      * From the main navigation menu, go to **APIs & Services -\> Library**.
      * Search for and **ENABLE** each of the following APIs one by one:
          * Vertex AI API
          * Cloud Functions API
          * Cloud Build API
          * Cloud Run Admin API
          * Cloud Firestore API
          * Identity and Access Management (IAM) API
          * BigQuery API
          * Google Maps Platform (This will group several APIs. Ensure the **Maps JavaScript API** and **Places API** are enabled within this group).

4.  **Create Your Maps API Key:**

      * In the **APIs & Services** section, go to the **Credentials** tab on the left.
      * Click **+ CREATE CREDENTIALS** at the top and select **API key**.
      * A key will be generated. Click **COPY** and save it securely in a text file for later. You'll need it for the frontend.

5.  **Set Up BigQuery for Analytics:**

      * In the GCP Console's main navigation menu, scroll down to the **ANALYTICS** section and click on **BigQuery**.
      * In the **Explorer** panel on the left, find and click on your `hackathon-trip-planner` project.
      * Click the three vertical dots next to your project name and select **Create dataset**.
          * **Dataset ID:** Enter `trip_analytics`.
          * **Location type:** Keep the default `Multi-region` or choose one that matches your function's location (e.g., `us-central1`).
          * Click **CREATE DATASET**.
      * Now, in the Explorer panel, find and expand your project to see the new `trip_analytics` dataset. Click the three vertical dots next to it and select **Create table**.
          * **Table name:** Enter `generated_trips`.
          * Find the **Schema** section. Toggle the **Edit as text** option.
          * Delete the existing content and paste the following JSON exactly:
            ```json
            [
              {"name": "trip_id", "type": "STRING", "mode": "REQUIRED"},
              {"name": "prompt", "type": "STRING", "mode": "NULLABLE"},
              {"name": "title", "type": "STRING", "mode": "NULLABLE"},
              {"name": "generated_at", "type": "TIMESTAMP", "mode": "REQUIRED"}
            ]
            ```
          * Click **CREATE TABLE**. Your analytics table is now ready to receive data.

6.  **Create and Configure Firebase Project:**

      * Go to the [Firebase Console](https://console.firebase.google.com/).
      * Click **Add project** and select your existing `hackathon-trip-planner` GCP project from the dropdown. Continue through the prompts.
      * **Upgrade your plan:** This is a critical step. On your main Firebase project page, look at the bottom of the left-hand menu. Click where it says **Spark Plan**. Select the **Blaze (Pay as you go)** plan. This is required to make external API calls (to Vertex AI), but you will remain well within the free tier for this project's scale.
      * **Set up Firestore:** Go to **Build -\> Firestore Database**. Click **Create database**. Start in **Production mode**. Choose a location for your data (e.g., `nam5 (us-central)`).
      * **Set up Authentication:** Go to **Build -\> Authentication**. Click **Get Started**. In the **Sign-in method** tab, select **Google** from the list of providers. Enable it, provide a project support email, and click **Save**.

-----

### **Milestone 2: Local Environment & The AI Backend**

**Objective:** To initialize your project on your local machine and code the complete, multi-purpose Firebase Function.

1.  **Install & Login with Firebase CLI:**

      * Open your computer's terminal (or Command Prompt).
      * Run `npm install -g firebase-tools` to get the latest tools.
      * Run `firebase login` and follow the prompts to log in with your Google account.

2.  **Initialize Your Local Project:**

      * Create a folder for your project on your computer and navigate into it:
        `mkdir my-trip-planner && cd my-trip-planner`
      * Run the initialization command:
        `firebase init`
      * When prompted, use the arrow keys and spacebar to select: **Firestore**, **Functions**, and **Hosting**. Press Enter.
      * Choose **Use an existing project** and select `hackathon-trip-planner` from the list.
      * **Firestore:** Press Enter to accept the default `firestore.rules` and `firestore.indexes.json` files.
      * **Functions:** Choose **JavaScript**. Answer **Y** to use ESLint. Answer **Y** to install dependencies with npm now.
      * **Hosting:**
          * What do you want to use as your public directory? Type `frontend/build`.
          * Configure as a single-page app (rewrite all urls to /index.html)? Answer **Y**.
          * Set up automatic builds and deploys with GitHub? Answer **N**.

3.  **Code the Backend Logic:**

      * Navigate into the functions directory: `cd functions`
      * Install the necessary SDKs in one command: `npm install @google-cloud/vertexai @google-cloud/bigquery uuid`
      * Open the `functions/index.js` file and **replace its entire contents** with the following fully-commented code:

    <!-- end list -->

    ````javascript
    // functions/index.js

    // Import necessary Firebase and Google Cloud modules
    const { onCall } = require("firebase-functions/v2/https");
    const { VertexAI } = require('@google-cloud/vertexai');
    const { BigQuery } = require('@google-cloud/bigquery');
    const { v4: uuidv4 } = require('uuid'); // To generate unique IDs
    const logger = require("firebase-functions/logger");

    // Initialize all Google Cloud clients with project details
    const vertex_ai = new VertexAI({ project: "hackathon-trip-planner", location: "us-central1" });
    const bigquery = new BigQuery();
    const model = 'gemini-1.5-pro-001'; // Using a powerful and recent model

    // The master prompt that instructs the AI. This is the core of your "unique flavor".
    const SYSTEM_PROMPT = `
    You are 'Roamer', an expert AI travel guide and storyteller. Your sole purpose is to create a captivating, narrative-driven travel itinerary.
    You MUST adhere to the following rules at all times:
    1.  Your entire response must be a single, valid JSON object. Do not include any text, markdown like \`\`\`json, or any other characters before or after the JSON object.
    2.  The JSON object must have this exact structure: { "tripTitle": "...", "summary": "...", "itinerary": [ { "day": 1, "theme": "...", "narrative": "...", "activities": [ { "time": "...", "title": "...", "description": "...", "locationName": "...", "location": { "lat": 12.345, "lng": 12.345 }, "estimatedCost": "..." } ] } ] }.
    3.  For the "location" object, you MUST provide plausible latitude and longitude coordinates for the corresponding "locationName".
    4.  Generate a complete, day-by-day itinerary based on the user's request. The narrative must be engaging and inspiring.
    `;

    // A secure, "Callable Function" that our frontend will invoke. This is better than a simple HTTP trigger.
    exports.generateItinerary = onCall(async (request) => {
      const userPrompt = request.data.prompt;

      // Validate that a prompt was actually sent
      if (!userPrompt) {
        throw new functions.https.HttpsError('invalid-argument', 'The function must be called with a "prompt" argument.');
      }
      logger.info(`Received prompt: ${userPrompt}`, { structuredData: true });

      // Construct the full prompt for the AI model
      const generativeModel = vertex_ai.getGenerativeModel({ model: model });
      const prompt = `${SYSTEM_PROMPT}\n\nUser Request: ${userPrompt}`;

      try {
        // Await the response from the Gemini model
        const resp = await generativeModel.generateContent(prompt);
        const content = resp.response.candidates[0].content.parts[0].text;
        
        // Clean the response to ensure it's a valid JSON string
        const jsonString = content.replace(/```json/g, "").replace(/```/g, "").trim();
        const itineraryData = JSON.parse(jsonString);

        // Asynchronously log metadata to BigQuery. This does not slow down the user response.
        const logDataToBigQuery = async () => {
          try {
            const row = {
              trip_id: uuidv4(),
              prompt: userPrompt,
              title: itineraryData.tripTitle,
              generated_at: new Date().toISOString(),
            };
            await bigquery.dataset('trip_analytics').table('generated_trips').insert([row]);
            logger.info("Successfully logged trip to BigQuery.");
          } catch (bqError) {
            logger.error("Failed to log to BigQuery:", bqError);
          }
        };
        logDataToBigQuery(); // Fire-and-forget this operation

        logger.info("Successfully generated itinerary.", { structuredData: true });
        // Return the successful result to the frontend
        return { status: "success", data: itineraryData };
      } catch (error) {
        logger.error("Error generating itinerary:", error);
        throw new functions.https.HttpsError('internal', 'Failed to generate itinerary. The AI may be overloaded or the response was malformed.', error);
      }
    });
    ````

4.  **Deploy the Backend:**

      * While still in the `functions` directory, run: `firebase deploy --only functions`
      * This will take a few minutes. Wait for the "Deploy complete\!" message.

-----

### **Milestone 3: The Frontend Application**

**Objective:** To build the complete user interface in React, including all interactive elements.

1.  **Create the React App:**

      * Navigate back to your project's root directory: `cd ..`
      * Run this command to create the frontend folder and app: `npx create-react-app frontend`

2.  **Install Frontend Dependencies:**

      * Navigate into the new frontend directory: `cd frontend`
      * Install the Firebase and Google Maps libraries: `npm install firebase @vis.gl/react-google-maps`

3.  **Connect Frontend to Firebase:**

      * In the Firebase Console, go to **Project Settings** (click the gear icon). Under the **General** tab, scroll down to **Your apps**.
      * Click the Web icon (`</>`). Register the app (e.g., "Trip Planner Web") and copy the `firebaseConfig` object it provides.
      * In your project, create a new file at `frontend/src/firebaseConfig.js`. Paste the following code, inserting your copied config object.

    <!-- end list -->

    ```javascript
    // frontend/src/firebaseConfig.js
    import { initializeApp } from "firebase/app";
    import { getFunctions, httpsCallable } from "firebase/functions";
    import { getAuth } from "firebase/auth";
    import { getFirestore } from "firebase/firestore";

    // PASTE YOUR FIREBASE CONFIG OBJECT FROM THE FIREBASE CONSOLE HERE
    const firebaseConfig = {
      apiKey: "AIza...",
      authDomain: "hackathon-trip-planner.firebaseapp.com",
      projectId: "hackathon-trip-planner",
      storageBucket: "hackathon-trip-planner.appspot.com",
      messagingSenderId: "...",
      appId: "..."
    };

    // Initialize Firebase and export the services
    const app = initializeApp(firebaseConfig);
    export const functions = getFunctions(app);
    export const auth = getAuth(app);
    export const db = getFirestore(app);

    // Create a callable reference to our backend function
    export const generateItineraryCallable = httpsCallable(functions, 'generateItinerary');
    ```

4.  **Create the Map Component:**

      * Create a new file at `frontend/src/MapView.js`. This component will render the map.

    <!-- end list -->

    ```javascript
    // frontend/src/MapView.js
    import React from 'react';
    import { APIProvider, Map, AdvancedMarker } from '@vis.gl/react-google-maps';

    const MapView = ({ itinerary }) => {
      // Default map center to a known location, or the first activity of the trip
      const centerPosition = itinerary?.itinerary[0]?.activities[0]?.location || { lat: 28.6139, lng: 77.2090 };
      const allActivities = itinerary?.itinerary.flatMap(day => day.activities) || [];

      return (
        <div className="map-container">
          {/* Be sure to replace the API Key below! */}
          <APIProvider apiKey="YOUR_GOOGLE_MAPS_API_KEY">
            <Map zoom={11} center={centerPosition} mapId="TRIP_MAP_ID">
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

      * **CRITICAL:** Replace `YOUR_GOOGLE_MAPS_API_KEY` with the key you generated in Milestone 1.

5.  **Create the Main App Component & Styling:**

      * Replace the contents of `frontend/src/App.js` with this complete file. It handles state, auth, API calls, and rendering.

    <!-- end list -->

    ```javascript
    // frontend/src/App.js
    import React, { useState, useEffect } from 'react';
    import { GoogleAuthProvider, signInWithPopup, onAuthStateChanged, signOut } from "firebase/auth";
    import { collection, addDoc, serverTimestamp } from "firebase/firestore";
    import { auth, db, generateItineraryCallable } from './firebaseConfig';
    import MapView from './MapView';
    import './App.css';

    function App() {
      const [prompt, setPrompt] = useState('');
      const [itinerary, setItinerary] = useState(null);
      const [loading, setLoading] = useState(false);
      const [error, setError] = useState('');
      const [user, setUser] = useState(null);

      // Effect to listen for changes in user authentication status
      useEffect(() => {
        const unsubscribe = onAuthStateChanged(auth, currentUser => setUser(currentUser));
        return () => unsubscribe(); // Cleanup on component unmount
      }, []);

      const handleGoogleLogin = async () => await signInWithPopup(auth, new GoogleAuthProvider());
      const handleLogout = async () => await signOut(auth);

      const handleSaveTrip = async () => {
        if (!user || !itinerary) return;
        try {
          await addDoc(collection(db, "users", user.uid, "trips"), { ...itinerary, savedAt: serverTimestamp() });
          alert("Trip saved to your profile!");
        } catch (err) { console.error("Error saving trip: ", err); alert("Failed to save trip."); }
      };

      const handleSubmit = async (e) => {
        e.preventDefault();
        if (!prompt) return;
        setLoading(true); setError(''); setItinerary(null);
        try {
          const result = await generateItineraryCallable({ prompt });
          if(result.data.status === 'success') {
            setItinerary(result.data.data);
          } else {
            throw new Error('API returned an error');
          }
        } catch (err) { setError('Failed to generate itinerary. The AI may be overloaded. Please try again.'); console.error(err); }
        finally { setLoading(false); }
      };

      return (
        <div className="App">
          <header className="App-header">
            <div className="auth-buttons">
              {user ? (
                <div className="user-info">
                  <img src={user.photoURL} alt={user.displayName} />
                  <span>{user.displayName}</span>
                  <button onClick={handleLogout} className="auth-btn">Logout</button>
                </div>
              ) : <button onClick={handleGoogleLogin} className="auth-btn">Sign in with Google</button>}
            </div>
            <h1>🚀 AI Storyteller Trip Planner</h1>
            <p>Describe your dream trip. We'll craft the perfect narrative and visual journey.</p>
          </header>

          <main>
            <form onSubmit={handleSubmit} className="prompt-form">
              <textarea value={prompt} onChange={(e) => setPrompt(e.target.value)} placeholder="e.g., A 5-day adventure in Rishikesh for two, focusing on yoga, river rafting, and local cafes. Medium budget." />
              <button type="submit" disabled={loading}>{loading ? 'Crafting Your Story...' : 'Generate Itinerary'}</button>
            </form>

            {error && <p className="error-message">{error}</p>}
            {itinerary && (
              <div className="results-container">
                <div className="itinerary-view">
                  <h2>{itinerary.tripTitle}</h2>
                  <p className="summary">{itinerary.summary}</p>
                  {user && <button onClick={handleSaveTrip} className="save-button">Save This Trip</button>}
                  {itinerary.itinerary.map((day) => (
                    <div key={day.day} className="day-card">
                      <h3>Day {day.day}: {day.theme}</h3>
                      <p className="narrative">{day.narrative}</p>
                      <ul className="activity-list">{day.activities.map((act, i) => <li key={i}><strong>{act.time} - {act.title}</strong>: {act.description}</li>)}</ul>
                    </div>
                  ))}
                </div>
                <MapView itinerary={itinerary} />
              </div>
            )}
          </main>
        </div>
      );
    }
    export default App;
    ```

      * Replace the contents of `frontend/src/App.css` with a better stylesheet to ensure your prototype looks polished. You can find many free CSS templates online, or use this as a starting point.

-----

### **Milestone 4: Final Deployment & The Pitch**

**Objective:** To deploy your application to the web and prepare the key talking points for your presentation.

1.  **Test Locally:**

      * From within the `frontend` directory, run `npm start`. Your app will open in a browser. Test every feature: login, generating a trip, seeing the map, saving the trip.

2.  **Build for Production:**

      * Once everything works, stop the local server (Ctrl+C). In the same `frontend` directory, run the build command:
        `npm run build`
      * This creates a final, optimized version of your app in the `frontend/build` folder, which you told Firebase to look for.

3.  **Deploy to the World:**

      * Navigate back to your project's root directory: `cd ..`
      * Run the final deploy command to push your frontend to the web:
        `firebase deploy --only hosting`
      * After it finishes, Firebase will give you a **Hosting URL**. This is the live link to your finished prototype.

#### **Your Winning Pitch Angle**

When you present, focus on these three pillars:

1.  **The Immersive Experience:** "Existing trip planners give you a to-do list. Our AI Storyteller, 'Roamer', gives you an *experience*. We craft a compelling narrative that builds excitement and emotional connection to the trip before it even begins." **(Demo the narrative output).**
2.  **Seamless Technology Integration:** "We've built this on a cutting-edge, serverless Google Cloud stack. This allows for infinite scalability and rapid feature development. From the advanced reasoning of **Vertex AI's Gemini model** to the real-time visualization on **Google Maps** and the secure, personalized profiles managed by **Firebase**, every piece works in concert." **(Show the architecture diagram).**
3.  **The Data-Driven Future:** "This is more than a planner; it's an intelligence engine. Every trip generated enriches our **BigQuery** dataset, allowing us to identify emerging travel trends, create hyper-personalized future recommendations, and uncover valuable market insights." **(Show your BigQuery table with the logged data).**

We now have a complete, deployed, and impressive prototype with a powerful story to tell.
