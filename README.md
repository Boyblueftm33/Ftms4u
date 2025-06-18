# Ftms4u
Ftm dating app 
import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, getDoc, setDoc, collection, query, where, onSnapshot, updateDoc, arrayUnion, arrayRemove, addDoc } from 'firebase/firestore';

// Define context for Firebase and User
const AppContext = createContext(null);

const AppProvider = ({ children }) => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [loadingFirebase, setLoadingFirebase] = useState(true);

    useEffect(() => {
        const initializeFirebase = async () => {
            try {
                const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
                const app = initializeApp(firebaseConfig);
                const firestoreDb = getFirestore(app);
                const firestoreAuth = getAuth(app);

                setDb(firestoreDb);
                setAuth(firestoreAuth);

                onAuthStateChanged(firestoreAuth, async (user) => {
                    if (user) {
                        setUserId(user.uid);
                    } else {
                        // Sign in anonymously if no user is authenticated and no token is provided
                        if (typeof __initial_auth_token === 'undefined') {
                            await signInAnonymously(firestoreAuth);
                        }
                    }
                    setLoadingFirebase(false);
                });

                // Sign in with custom token if available
                if (typeof __initial_auth_token !== 'undefined') {
                    await signInWithCustomToken(firestoreAuth, __initial_auth_token);
                }

            } catch (error) {
                console.error("Error initializing Firebase:", error);
                setLoadingFirebase(false);
            }
        };

        initializeFirebase();
    }, []);

    return (
        <AppContext.Provider value={{ db, auth, userId, loadingFirebase }}>
            {children}
        </AppContext.Provider>
    );
};


const App = () => {
    const { db, userId, loadingFirebase } = useContext(AppContext);
    const [currentPage, setCurrentPage] = useState('profile'); // 'profile', 'browse', 'messages'
    const [userProfile, setUserProfile] = useState(null);
    const [message, setMessage] = useState('');
    const [showProfileSetup, setShowProfileSetup] = useState(false);
    const [profileFormData, setProfileFormData] = useState({ name: '', bio: '', genderIdentity: 'FTM', lookingFor: 'Cisgender' });
    const [browseProfiles, setBrowseProfiles] = useState([]);
    const [matchedUsers, setMatchedUsers] = useState([]);
    const [selectedChatUser, setSelectedChatUser] = useState(null);
    const [chatMessages, setChatMessages] = useState([]);
    const [showModal, setShowModal] = useState(false);
    const [modalContent, setModalContent] = useState('');
    const [generatingBio, setGeneratingBio] = useState(false); // New state for bio generation loading
    const [generatingIcebreakers, setGeneratingIcebreakers] = useState(false); // New state for icebreaker generation loading
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';


    useEffect(() => {
        if (!loadingFirebase && userId && db) {
            // Fetch user profile
            const userDocRef = doc(db, `artifacts/${appId}/users/${userId}/profiles`, userId);
            const unsubscribeProfile = onSnapshot(userDocRef, (docSnap) => {
                if (docSnap.exists()) {
                    setUserProfile(docSnap.data());
                    setShowProfileSetup(false); // Hide setup if profile exists
                    setProfileFormData(docSnap.data()); // Populate form data with existing profile
                } else {
                    setUserProfile(null);
                    setShowProfileSetup(true); // Show setup if no profile
                }
            }, (error) => {
                console.error("Error fetching user profile:", error);
                setMessage('Error fetching user profile.');
            });

            // Fetch matched users
            const matchesCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/matches`);
            const unsubscribeMatches = onSnapshot(matchesCollectionRef, (snapshot) => {
                const matches = snapshot.docs.map(doc => doc.data());
                setMatchedUsers(matches);
            }, (error) => {
                console.error("Error fetching matches:", error);
                setMessage('Error fetching matches.');
            });

            return () => {
                unsubscribeProfile();
                unsubscribeMatches();
            };
        }
    }, [loadingFirebase, userId, db, appId]);


    useEffect(() => {
        if (!loadingFirebase && db && userProfile && currentPage === 'browse') {
            // Fetch potential profiles for browsing
            const profilesRef = collection(db, `artifacts/${appId}/public/data/profiles`);
            const q = query(profilesRef);

            const unsubscribe = onSnapshot(q, (snapshot) => {
                const profiles = snapshot.docs
                    .map(doc => ({ id: doc.id, ...doc.data() }))
                    .filter(profile => profile.id !== userId); // Don't show current user's profile

                // Filter out profiles that have already been liked or matched
                const filteredProfiles = profiles.filter(profile => {
                    const isMatched = matchedUsers.some(match => match.matchedUserId === profile.id);
                    // For a real app, you'd also check if they've already been "passed" or "liked"
                    return !isMatched;
                });
                setBrowseProfiles(filteredProfiles);
            }, (error) => {
                console.error("Error fetching browse profiles:", error);
                setMessage('Error fetching browse profiles.');
            });

            return () => unsubscribe();
        }
    }, [loadingFirebase, db, userId, currentPage, userProfile, matchedUsers, appId]);

    // Effect for fetching chat messages
    useEffect(() => {
        if (db && userId && selectedChatUser) {
            const chatCollectionId = [userId, selectedChatUser.matchedUserId].sort().join('_');
            const messagesRef = collection(db, `artifacts/${appId}/public/data/chats/${chatCollectionId}/messages`);
            const unsubscribe = onSnapshot(messagesRef, (snapshot) => {
                const messages = snapshot.docs.map(doc => doc.data()).sort((a, b) => a.timestamp - b.timestamp);
                setChatMessages(messages);
            }, (error) => {
                console.error("Error fetching chat messages:", error);
            });
            return () => unsubscribe();
        }
    }, [db, userId, selectedChatUser, appId]);


    const handleProfileFormChange = (e) => {
        const { name, value } = e.target;
        setProfileFormData(prev => ({ ...prev, [name]: value }));
    };

    const handleProfileSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) {
            setModalContent('Firebase not ready. Please try again.');
            setShowModal(true);
            return;
        }

        try {
            // Save profile to user's private collection
            const userProfileDocRef = doc(db, `artifacts/${appId}/users/${userId}/profiles`, userId);
            await setDoc(userProfileDocRef, { ...profileFormData, userId: userId });

            // Also save to public collection for browsing by others
            const publicProfileDocRef = doc(db, `artifacts/${appId}/public/data/profiles`, userId);
            await setDoc(publicProfileDocRef, { ...profileFormData, userId: userId });

            setMessage('Profile saved successfully!');
            setUserProfile({ ...profileFormData, userId: userId });
            setShowProfileSetup(false);
            setCurrentPage('browse'); // Navigate to browse after profile setup
        } catch (error) {
            console.error("Error saving profile:", error);
            setMessage('Failed to save profile.');
            setModalContent(`Failed to save profile: ${error.message}`);
            setShowModal(true);
        }
    };

    const handleLike = async (profileId, profileName) => {
        if (!db || !userId) return;

        try {
            // Add to current user's matches collection
            const currentUserMatchesRef = doc(db, `artifacts/${appId}/users/${userId}/matches`, profileId);
            await setDoc(currentUserMatchesRef, { matchedUserId: profileId, name: profileName, status: 'liked' });

            // Check if the other user also liked the current user (mutual match)
            const otherUserMatchesRef = doc(db, `artifacts/${appId}/users/${profileId}/matches`, userId);
            const otherUserMatchSnap = await getDoc(otherUserMatchesRef);

            if (otherUserMatchSnap.exists() && otherUserMatchSnap.data().status === 'liked') {
                // Mutual match! Update both sides to 'matched'
                await updateDoc(currentUserMatchesRef, { status: 'matched' });
                await updateDoc(otherUserMatchesRef, { status: 'matched' });
                setModalContent(`It's a Match with ${profileName}!`);
                setShowModal(true);
            } else {
                setModalContent(`You liked ${profileName}. Waiting for them to like you back!`);
                setShowModal(true);
            }
            // Remove the liked profile from the browse list
            setBrowseProfiles(prev => prev.filter(p => p.id !== profileId));
        } catch (error) {
            console.error("Error liking profile:", error);
            setMessage('Failed to like profile.');
            setModalContent(`Failed to like profile: ${error.message}`);
            setShowModal(true);
        }
    };

    const handlePass = (profileId) => {
        // In a real app, you might record this so they don't appear again
        setBrowseProfiles(prev => prev.filter(p => p.id !== profileId));
        setMessage('Profile passed.');
    };

    const handleSendMessage = async (e) => {
        e.preventDefault();
        if (!db || !userId || !selectedChatUser || !message.trim()) return;

        const chatCollectionId = [userId, selectedChatUser.matchedUserId].sort().join('_');
        const messagesRef = collection(db, `artifacts/${appId}/public/data/chats/${chatCollectionId}/messages`);

        try {
            await addDoc(messagesRef, {
                senderId: userId,
                senderName: userProfile?.name || 'Anonymous',
                receiverId: selectedChatUser.matchedUserId,
                text: message.trim(),
                timestamp: Date.now(),
            });
            setMessage('');
        } catch (error) {
            console.error("Error sending message:", error);
            setModalContent(`Failed to send message: ${error.message}`);
            setShowModal(true);
        }
    };

    const closeModal = () => {
        setShowModal(false);
        setModalContent('');
    };

    // New function for generating bio
    const handleGenerateBio = async () => {
        setGeneratingBio(true);
        try {
            const prompt = `Write a compelling and concise dating app bio for a person named ${profileFormData.name || 'someone'} who identifies as ${profileFormData.genderIdentity}. Here's some existing info: "${profileFormData.bio}". Make it engaging and positive, focusing on connecting with the ${profileFormData.lookingFor} population. Keep it under 200 characters.`;

            let chatHistory = [];
            chatHistory.push({ role: "user", parts: [{ text: prompt }] });
            const payload = { contents: chatHistory };
            const apiKey = ""; // If you want to use models other than gemini-2.0-flash or imagen-3.0-generate-002, provide an API key here. Otherwise, leave this as-is.
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            const result = await response.json();
            if (result.candidates && result.candidates.length > 0 &&
                result.candidates[0].content && result.candidates[0].content.parts &&
                result.candidates[0].content.parts.length > 0) {
                const generatedText = result.candidates[0].content.parts[0].text;
                setProfileFormData(prev => ({ ...prev, bio: generatedText.trim() }));
                setMessage('Bio generated successfully!');
            } else {
                console.error("Gemini API response structure unexpected:", result);
                setModalContent('Could not generate bio. Please try again.');
                setShowModal(true);
            }
        } catch (error) {
            console.error("Error calling Gemini API for bio generation:", error);
            setModalContent(`Error generating bio: ${error.message}`);
            setShowModal(true);
        } finally {
            setGeneratingBio(false);
        }
    };

    // New function for generating icebreakers
    const handleGenerateIcebreakers = async () => {
        if (!selectedChatUser) return;
        setGeneratingIcebreakers(true);
        try {
            const prompt = `Suggest 3-5 engaging and friendly icebreaker questions for a dating app conversation. The person you are trying to talk to is named ${selectedChatUser.name} and their bio says: "${selectedChatUser.bio}". Make the questions personalized and open-ended.`;

            let chatHistory = [];
            chatHistory.push({ role: "user", parts: [{ text: prompt }] });
            const payload = { contents: chatHistory };
            const apiKey = ""; // If you want to use models other than gemini-2.0-flash or imagen-3.0-generate-002, provide an API key here. Otherwise, leave this as-is.
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            const result = await response.json();
            if (result.candidates && result.candidates.length > 0 &&
                result.candidates[0].content && result.candidates[0].content.parts &&
                result.candidates[0].content.parts.length > 0) {
                const generatedText = result.candidates[0].content.parts[0].text;
                setModalContent(`Suggested Icebreakers for ${selectedChatUser.name}:\n\n${generatedText}`);
                setShowModal(true);
            } else {
                console.error("Gemini API response structure unexpected:", result);
                setModalContent('Could not suggest icebreakers. Please try again.');
                setShowModal(true);
            }
        } catch (error) {
            console.error("Error calling Gemini API for icebreaker generation:", error);
            setModalContent(`Error suggesting icebreakers: ${error.message}`);
            setShowModal(true);
        } finally {
            setGeneratingIcebreakers(false);
        }
    };


    if (loadingFirebase) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100">
                <div className="text-xl text-gray-700">Loading app...</div>
            </div>
        );
    }

    return (
        <div className="min-h-screen bg-gradient-to-br from-purple-200 to-pink-300 font-inter text-gray-800 p-4 flex flex-col items-center">
            <header className="w-full max-w-4xl bg-white bg-opacity-80 rounded-xl shadow-lg p-4 mb-6 flex justify-between items-center">
                <h1 className="text-3xl font-bold text-purple-700">LoveBridge</h1>
                <nav className="flex space-x-4">
                    <button
                        onClick={() => setCurrentPage('profile')}
                        className={`px-4 py-2 rounded-lg font-medium transition-colors ${currentPage === 'profile' ? 'bg-purple-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-purple-100'}`}
                    >
                        My Profile
                    </button>
                    <button
                        onClick={() => setCurrentPage('browse')}
                        className={`px-4 py-2 rounded-lg font-medium transition-colors ${currentPage === 'browse' ? 'bg-purple-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-purple-100'}`}
                    >
                        Browse
                    </button>
                    <button
                        onClick={() => setCurrentPage('messages')}
                        className={`px-4 py-2 rounded-lg font-medium transition-colors ${currentPage === 'messages' ? 'bg-purple-600 text-white shadow-md' : 'bg-gray-200 text-gray-700 hover:bg-purple-100'}`}
                    >
                        Messages ({matchedUsers.filter(m => m.status === 'matched').length})
                    </button>
                </nav>
            </header>

            <main className="w-full max-w-4xl bg-white bg-opacity-90 rounded-xl shadow-xl p-6">
                {message && (
                    <div className="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded-md relative mb-4">
                        {message}
                    </div>
                )}

                {userId && (
                    <div className="text-sm text-gray-600 mb-4 text-center">
                        Your User ID: <span className="font-mono bg-gray-100 p-1 rounded">{userId}</span>
                    </div>
                )}


                {showProfileSetup && currentPage === 'profile' && (
                    <div className="p-6 bg-purple-50 rounded-lg shadow-inner">
                        <h2 className="text-2xl font-semibold mb-4 text-purple-800">Set Up Your Profile</h2>
                        <form onSubmit={handleProfileSubmit} className="space-y-4">
                            <div>
                                <label htmlFor="name" className="block text-sm font-medium text-gray-700">Name</label>
                                <input
                                    type="text"
                                    id="name"
                                    name="name"
                                    value={profileFormData.name}
                                    onChange={handleProfileFormChange}
                                    required
                                    className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-purple-500 focus:border-purple-500 sm:text-sm"
                                />
                            </div>
                            <div>
                                <label htmlFor="bio" className="block text-sm font-medium text-gray-700">Bio</label>
                                <textarea
                                    id="bio"
                                    name="bio"
                                    value={profileFormData.bio}
                                    onChange={handleProfileFormChange}
                                    rows="3"
                                    className="mt-1 block w-full px-3 py-2 border border-g
