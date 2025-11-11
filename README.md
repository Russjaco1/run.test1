<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Grocery Price Tracker & Regional Filter</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Load ZXing-JS library for barcode scanning -->
    <script src="https://unpkg.com/@zxing/library@0.20.0/umd/index.min.js"></script>
    <!-- Use Inter Font -->
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7fafc; /* Light gray background */
        }
        .container {
            max-width: 1200px; 
        }
        /* Custom scrollbar style for the filter section */
        #store-filter-container {
            max-height: 150px;
            overflow-y: auto;
        }
        /* General modal styles */
        .modal {
            transition: opacity 0.3s ease, visibility 0.3s ease;
        }
        .modal:not(.open) {
            opacity: 0;
            visibility: hidden;
        }
        /* Hide scrollbar when sidebar is open */
        .overflow-hidden {
            overflow: hidden;
        }
        /* Scanner specific styles */
        #scanner-video {
            width: 100%;
            height: auto;
            max-height: 80vh;
            border-radius: 1rem;
            object-fit: cover;
            transform: scaleX(-1); /* Mirror effect for better UX */
        }
        #scan-box {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 80%;
            max-width: 400px;
            height: 25vh;
            border: 2px dashed #4F46E5; /* Indigo */
            pointer-events: none;
            box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.5); /* Dim surrounding area */
            z-index: 10;
            border-radius: 0.5rem;
        }
    </style>
    <!-- Firebase Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, serverTimestamp, setLogLevel, getDocs, deleteDoc, where } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase variables will be attached to the window object for easy access
        window.firebaseApp = null;
        window.db = null;
        window.auth = null;
        window.userId = 'unknown';

        // Global data storage for comparison and filtering
        window.allPricesMap = {}; 
        window.uniqueStores = []; 
        window.activeStoreFilters = new Set(); 
        window.searchTerm = ''; 
        window.shopCart = new Set(); 
        window.activeRegionCode = null; 

        // NEW: Barcode Scanner State
        let codeReader = null;
        let videoElement = null;
        let currentStream = null;


        // Set log level for debugging
        setLogLevel('Debug');

        // Global variables provided by the environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        const PUBLIC_PRICES_COLLECTION = `artifacts/${appId}/public/data/prices`;

        /**
         * Generic custom modal/confirmation dialog.
         * @param {string} title - The title of the modal.
         * @param {string} message - The message body.
         * @param {string} confirmText - Text for the confirm button.
         * @param {string} confirmClass - Tailwind classes for the confirm button (e.g., 'bg-red-600').
         * @param {boolean} showCancel - Whether to show the cancel button (defaults to true).
         * @returns {Promise<boolean>} Resolves to true if confirmed, false if canceled.
         */
        window.showCustomModal = function(title, message, confirmText, confirmClass, showCancel = true) {
            return new Promise((resolve) => {
                const modal = document.getElementById('custom-confirm-modal');
                const titleEl = document.getElementById('custom-modal-title');
                const messageEl = document.getElementById('custom-modal-message');
                const confirmBtn = document.getElementById('custom-modal-confirm');
                const cancelBtn = document.getElementById('custom-modal-cancel');

                titleEl.textContent = title;
                messageEl.textContent = message;
                confirmBtn.textContent = confirmText;
                
                // Reset classes and apply new one
                confirmBtn.className = 'px-6 py-2 text-white rounded-lg font-semibold transition ' + confirmClass;
                
                if (showCancel) {
                    cancelBtn.classList.remove('hidden');
                } else {
                    cancelBtn.classList.add('hidden');
                }

                modal.classList.add('open');
                document.body.classList.add('overflow-hidden');

                const handleConfirm = () => {
                    modal.classList.remove('open');
                    document.body.classList.remove('overflow-hidden');
                    confirmBtn.removeEventListener('click', handleConfirm);
                    cancelBtn.removeEventListener('click', handleCancel);
                    resolve(true);
                };

                const handleCancel = () => {
                    modal.classList.remove('open');
                    document.body.classList.remove('overflow-hidden');
                    confirmBtn.removeEventListener('click', handleConfirm);
                    cancelBtn.removeEventListener('click', handleCancel);
                    resolve(false);
                };

                confirmBtn.addEventListener('click', handleConfirm);
                cancelBtn.addEventListener('click', handleCancel);
            });
        }

        /**
         * Initializes Firebase and authenticates the user.
         */
        async function initializeAndAuthenticate() {
            try {
                if (Object.keys(firebaseConfig).length === 0) {
                    console.error("Firebase config is missing or invalid.");
                    document.getElementById('status-message').textContent = "ERROR: Firebase is not configured.";
                    return;
                }

                window.firebaseApp = initializeApp(firebaseConfig);
                window.db = getFirestore(window.firebaseApp);
                window.auth = getAuth(window.firebaseApp);

                // Authentication Handler
                onAuthStateChanged(window.auth, (user) => {
                    if (user) {
                        window.userId = user.uid;
                        document.getElementById('user-id-display').textContent = `User ID: ${window.userId}`;
                        console.log("Authentication successful, starting data listener...");
                        window.loadPrices(); // Start loading data once authenticated
                    } else {
                        console.warn("User signed out or authentication failed.");
                        window.userId = 'anonymous';
                    }
                });

                // Initial sign-in attempt
                if (initialAuthToken) {
                    await signInWithCustomToken(window.auth, initialAuthToken);
                    console.log("Signed in with custom token.");
                } else {
                    await signInAnonymously(window.auth);
                    console.log("Signed in anonymously.");
                }

                document.getElementById('status-message').textContent = "App Ready! Start logging prices or seed the database.";
                window.checkIfSeedingNeeded();

            } catch (error) {
                console.error("Error during Firebase initialization or authentication:", error);
                document.getElementById('status-message').textContent = `ERROR: Failed to initialize app (${error.code}). Check console.`;
            }
        }

        // --- Sidebar Logic (UNCHANGED) ---

        /**
         * Toggles the visibility of the sidebar menu.
         */
        window.toggleSidebar = function() {
            const sidebar = document.getElementById('sidebar');
            const backdrop = document.getElementById('sidebar-backdrop');
            
            if (sidebar.classList.contains('-translate-x-full')) {
                // Open sidebar
                sidebar.classList.remove('-translate-x-full');
                backdrop.classList.remove('opacity-0', 'pointer-events-none');
                backdrop.classList.add('opacity-50');
                document.body.classList.add('overflow-hidden'); // Prevent scrolling content behind
            } else {
                // Close sidebar
                sidebar.classList.add('-translate-x-full');
                backdrop.classList.add('opacity-0', 'pointer-events-none');
                backdrop.classList.remove('opacity-50');
                document.body.classList.remove('overflow-hidden');
            }
        }
        
        /**
         * Handles actions for menu items (currently just closing the menu).
         */
        window.handleMenuAction = function(action) {
            console.log(`Menu action triggered: ${action}`);
            window.toggleSidebar(); // Close menu after action
            
            // Show a quick message to confirm the action
            const statusEl = document.getElementById('status-message');
            statusEl.textContent = `Action: '${action}' executed.`;
            setTimeout(() => statusEl.textContent = "App Ready! Start logging prices or seed the database.", 2000);
        }

        // --- Barcode Scanner Logic (NEW) ---

        /**
         * Opens the scanner modal and initializes the camera.
         */
        window.openScannerModal = function() {
            if (!window.db || !window.userId) {
                document.getElementById('form-error').textContent = "Error: Database not ready.";
                return;
            }

            document.getElementById('scanner-modal').classList.add('open');
            document.body.classList.add('overflow-hidden');
            
            videoElement = document.getElementById('scanner-video');
            codeReader = new ZXing.BrowserMultiFormatReader();
            
            // Set the feedback text
            document.getElementById('scanner-feedback').textContent = "Initializing camera and searching for barcode...";

            codeReader.getVideoInputDevices().then((videoInputDevices) => {
                const backCamera = videoInputDevices.find(device => device.label.toLowerCase().includes('back'));
                const selectedDeviceId = backCamera ? backCamera.deviceId : (videoInputDevices.length > 0 ? videoInputDevices[0].deviceId : null);
                
                if (selectedDeviceId) {
                    // Start decoding from the camera feed
                    codeReader.decodeFromVideoDevice(selectedDeviceId, videoElement, (result, err) => {
                        if (result) {
                            console.log('Scan Successful:', result.text);
                            // Stop the scanner and integrate the result
                            window.closeScannerModal(result.text);
                        }
                        if (err && !(err instanceof ZXing.NotFoundException)) {
                            // Only log non-NotFound errors
                            console.error('Scanning error:', err);
                            document.getElementById('scanner-feedback').textContent = `Error during scan: ${err.message}`;
                        }
                    });

                    // Store the current stream for stopping later
                    videoElement.onloadedmetadata = () => {
                        currentStream = videoElement.srcObject;
                        document.getElementById('scanner-feedback').textContent = "Camera ready. Center a barcode in the dashed box.";
                    };
                } else {
                    document.getElementById('scanner-feedback').textContent = "No camera found or permissions denied.";
                }
            }).catch((err) => {
                console.error('Camera access error:', err);
                document.getElementById('scanner-feedback').textContent = `Could not access camera: ${err.message}. Check permissions.`;
            });
        };

        /**
         * Closes the scanner modal and stops the camera stream.
         * @param {string} barcodeResult - The scanned barcode data (if successful).
         */
        window.closeScannerModal = function(barcodeResult = null) {
            // 1. Stop the camera stream and decoding
            if (codeReader) {
                codeReader.reset();
                codeReader = null;
            }
            if (currentStream) {
                currentStream.getTracks().forEach(track => track.stop());
                currentStream = null;
            }
            
            // 2. Close the modal UI
            document.getElementById('scanner-modal').classList.remove('open');
            document.body.classList.remove('overflow-hidden');
            
            // 3. Populate form if result is available
            if (barcodeResult) {
                // We use the barcode as the standardized item name for tracking
                const uniqueItemName = `[UPC: ${barcodeResult}]`; 
                
                document.getElementById('item-name').value = uniqueItemName;
                document.getElementById('item-price').focus();
                
                // Show success message on the main form
                document.getElementById('form-error').textContent = `Barcode scanned successfully! Item: ${uniqueItemName}. Now enter price and store.`;
                setTimeout(() => document.getElementById('form-error').textContent = "", 4000);
            }
        };


        // --- Data Seeding Logic (UNCHANGED) ---
        
        const SEED_DATA = [
            // Region 9 (West Coast - Higher Price Simulation)
            { itemName: 'Milk (Gallon)', store: 'Walmart', price: 3.99, zipCode: '90210' },
            { itemName: 'Bananas (1 lb)', store: 'Aldi', price: 0.59, zipCode: '90210' },
            { itemName: 'Butter (1 lb)', store: 'Target', price: 5.50, zipCode: '90210' },
            // Region 1 (Northeast - Medium-High Price Simulation)
            { itemName: 'Milk (Gallon)', store: 'Target', price: 4.99, zipCode: '10001' },
            { itemName: 'Chicken Breast (1 lb)', store: 'Target', price: 7.99, zipCode: '10001' },
            { itemName: 'Cereal (Box)', store: 'Target', price: 4.50, zipCode: '10001' },
            // Region 4 (Mid-South - Standard Price Simulation)
            { itemName: 'Milk (Gallon)', store: 'Kroger', price: 4.29, zipCode: '40203' },
            { itemName: 'Eggs (Dozen)', store: 'Kroger', price: 2.50, zipCode: '40203' },
            { itemName: 'Chicken Breast (1 lb)', store: 'Kroger', price: 6.80, zipCode: '40203' },
            { itemName: 'Cereal (Box)', store: 'Walmart', price: 3.99, zipCode: '40203' },
            // Region 7 (Southwest - Medium Price Simulation)
            { itemName: 'Eggs (Dozen)', store: 'Walmart', price: 2.99, zipCode: '78701' },
            { itemName: 'Bananas (1 lb)', store: 'Walmart', price: 0.65, zipCode: '78701' },
        ];


        /**
         * Checks if the collection has any documents and shows the seed button if it's empty.
         */
        window.checkIfSeedingNeeded = async function() {
            if (!window.db) return;
            try {
                const q = query(collection(window.db, PUBLIC_PRICES_COLLECTION));
                const snapshot = await getDocs(q);
                
                const seedButtonContainer = document.getElementById('seed-button-container');
                if (snapshot.empty) {
                    seedButtonContainer.classList.remove('hidden');
                } else {
                    seedButtonContainer.classList.add('hidden');
                }
            } catch (error) {
                console.error("Error checking for seed data:", error);
            }
        }

        window.showSeedModal = function() {
            document.getElementById('seed-modal').classList.add('open');
            document.getElementById('seed-status').textContent = `Ready to load ${SEED_DATA.length} sample price entries into the database. This will help demonstrate the regional filtering feature.`;
        }

        window.closeSeedModal = function() {
            document.getElementById('seed-modal').classList.remove('open');
        }
        
        /**
         * Executes the database seeding with mock data.
         */
        window.seedDatabase = async function() {
            if (!window.db || !window.userId) return;

            const seedBtn = document.getElementById('seed-confirm-btn');
            seedBtn.disabled = true;
            seedBtn.textContent = 'Seeding...';
            document.getElementById('seed-status').textContent = "Adding data entries. Please wait...";

            try {
                for (const item of SEED_DATA) {
                    const normalizedItemName = item.itemName.trim().charAt(0).toUpperCase() + item.itemName.trim().slice(1).toLowerCase();
                    const normalizedStoreName = item.store.trim().charAt(0).toUpperCase() + item.store.trim().slice(1).toLowerCase();
                    const regionCode = item.zipCode.charAt(0);
                    
                    await addDoc(collection(window.db, PUBLIC_PRICES_COLLECTION), {
                        itemName: normalizedItemName,
                        store: normalizedStoreName,
                        price: parseFloat(item.price.toFixed(2)),
                        timestamp: serverTimestamp(),
                        userId: window.userId,
                        regionCode: regionCode,
                        isScannedPrice: false // Seeded data is not scanned
                    });
                }
                
                document.getElementById('seed-status').textContent = `Successfully added ${SEED_DATA.length} sample entries!`;
                
                // Let the onSnapshot listener refresh the data, then close the modal
                setTimeout(() => {
                    window.closeSeedModal();
                    document.getElementById('seed-button-container').classList.add('hidden');
                }, 1500);

            } catch (e) {
                console.error("Error seeding database: ", e);
                document.getElementById('seed-status').textContent = `An error occurred during seeding: ${e.message}`;
            } finally {
                seedBtn.disabled = false;
                seedBtn.textContent = 'Proceed and Seed';
            }
        }


        // --- Cart Logic (UNCHANGED) ---

        /**
         * Adds or removes an item from the shop cart and updates the UI.
         */
        window.toggleCart = function(itemName) {
            if (window.shopCart.has(itemName)) {
                window.shopCart.delete(itemName);
            } else {
                window.shopCart.add(itemName);
            }
            window.renderComparisonList();      
            window.renderShopCartSummary();    
        }

        /**
         * Calculates and renders the total cost of the items in the shopCart for each store.
         */
        window.renderShopCartSummary = function() {
            const cartItemListEl = document.getElementById('cart-item-list');
            const cartSummaryTotalsEl = document.getElementById('cart-summary-totals');
            const cartItemCountEl = document.getElementById('cart-item-count');

            cartItemListEl.innerHTML = '';
            cartSummaryTotalsEl.innerHTML = '';
            
            // Get the list of items currently being displayed in the comparison list (after filtering)
            const filteredItems = Object.keys(window.allPricesMap).filter(itemName => 
                itemName.toLowerCase().includes(window.searchTerm)
            );

            // Filter the shop cart to only include items currently shown in the comparison list
            const activeCartItems = new Set([...window.shopCart].filter(item => filteredItems.includes(item)));
            
            cartItemCountEl.textContent = `(${activeCartItems.size} items)`;

            if (activeCartItems.size === 0) {
                cartItemListEl.innerHTML = '<p class="text-sm text-gray-500 italic p-3 text-center">Your cart is empty or items are filtered out.</p>';
                cartSummaryTotalsEl.innerHTML = '<p class="text-sm text-gray-500 italic p-3 text-center">Add items to compare store totals.</p>';
                return;
            }

            const storeTotals = {};
            const missingItemCounts = {};

            window.uniqueStores.forEach(store => {
                storeTotals[store] = 0;
                missingItemCounts[store] = 0;
            });
            
            activeCartItems.forEach(itemName => {
                const itemData = window.allPricesMap[itemName];

                // List item for the cart item list (allows removal)
                const listItem = document.createElement('li');
                listItem.className = 'flex justify-between items-center p-2 border-b last:border-b-0 text-sm';
                listItem.innerHTML = `
                    <span class="font-medium flex-1 truncate">${itemName}</span>
                    <button onclick="window.toggleCart('${itemName.replace(/'/g, "\\'")}')" 
                            class="text-red-500 hover:text-red-700 p-1 rounded-full bg-red-100 transition" 
                            title="Remove from Cart">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                          <path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                        </svg>
                    </button>
                `;
                cartItemListEl.appendChild(listItem);
                
                // Aggregate store totals based on the filtered data in allPricesMap
                window.uniqueStores.forEach(store => {
                    // Check entriesByStore from the aggregated data (which is filtered by region)
                    const priceEntry = itemData.entriesByStore[store]; 
                    if (priceEntry) {
                        storeTotals[store] += priceEntry.latestPrice;
                    } else {
                        missingItemCounts[store]++;
                    }
                });
            });

            const sortedStores = window.uniqueStores.sort((a, b) => {
                const totalA = storeTotals[a];
                const totalB = storeTotals[b];
                const missingA = missingItemCounts[a];
                const missingB = missingItemCounts[b];

                if (missingA > 0 && missingB === 0) return 1;
                if (missingA === 0 && missingB > 0) return -1;
                return totalA - totalB;
            });

            let bestStoreName = null;
            let cheapestPrice = Infinity;
            let allTotalsHtml = '';

            sortedStores.forEach(store => {
                const total = storeTotals[store];
                const missing = missingItemCounts[store];

                const isComplete = missing === 0;
                let priceDisplay = isComplete ? `$${total.toFixed(2)}` : 'N/A';
                let subText = isComplete ? 'Complete Cart Price' : `Missing ${missing} item(s)`;
                
                if (isComplete && total < cheapestPrice) {
                    cheapestPrice = total;
                    bestStoreName = store;
                }
                
                const isBest = store === bestStoreName && isComplete;
                const priceStyle = isBest ? 'text-indigo-600' : (isComplete ? 'text-gray-800' : 'text-yellow-600');
                const textStyle = isBest ? 'font-extrabold text-xl' : 'font-semibold text-lg';

                allTotalsHtml += `
                    <div class="flex justify-between text-base border-b border-gray-100 last:border-b-0 py-2">
                        <div class="flex flex-col">
                            <span class="font-semibold ${isBest ? 'text-indigo-600' : 'text-gray-700'}">${store}</span>
                            <span class="text-xs text-gray-500">${subText}</span>
                        </div>
                        <span class="${textStyle} ${priceStyle}">
                            ${priceDisplay}
                        </span>
                    </div>
                `;
            });

            cartSummaryTotalsEl.innerHTML = allTotalsHtml;
            
            if (bestStoreName) {
                 const finalSummaryEl = document.createElement('div');
                 finalSummaryEl.className = 'pt-3 mt-2 border-t border-indigo-200 flex flex-col justify-between font-extrabold text-lg';
                 finalSummaryEl.innerHTML = `
                    <span class="text-sm font-semibold text-gray-600">Cheapest Complete Cart:</span>
                    <span class="text-indigo-700 text-2xl mt-1">$${cheapestPrice.toFixed(2)}</span>
                    <span class="text-base text-gray-500 font-medium">at ${bestStoreName}</span>
                 `;
                 cartSummaryTotalsEl.appendChild(finalSummaryEl);
            } else if (activeCartItems.size > 0) {
                cartSummaryTotalsEl.innerHTML = '<p class="text-sm text-red-500 italic p-3 text-center">No complete cart found in the current filtered view.</p>';
            }
        }


        // --- Icon Helper Function (UNCHANGED) ---

        /**
         * Returns a relevant emoji based on the item name for visual representation.
         */
        function getItemIcon(itemName) {
            const lowerName = itemName.toLowerCase();
            if (lowerName.includes('[upc')) return '🏷️'; // New icon for barcode scanned items
            if (lowerName.includes('apple') || lowerName.includes('fruit')) return '🍎';
            if (lowerName.includes('milk') || lowerName.includes('gallon')) return '🥛';
            if (lowerName.includes('chicken') || lowerName.includes('turkey')) return '🍗';
            if (lowerName.includes('chip') || lowerName.includes('cookie')) return '🍪';
            if (lowerName.includes('butter')) return '🧈';
            if (lowerName.includes('egg')) return '🥚';
            if (lowerName.includes('cereal')) return '🥣';
            if (lowerName.includes('banana')) return '🍌';
            return '🛍️'; 
        }


        // --- Firestore Operations and Logic ---

        /**
         * Saves a new price entry to the Firestore database.
         */
        window.savePrice = async function() {
            if (!window.db || !window.userId) {
                document.getElementById('form-error').textContent = "Error: Database not ready. Please wait or refresh.";
                return;
            }

            const itemName = document.getElementById('item-name').value;
            const store = document.getElementById('store-name').value;
            const price = parseFloat(document.getElementById('item-price').value);
            const zipCode = document.getElementById('log-zip-code').value; 

            // Validation
            if (!itemName || !store || isNaN(price) || price <= 0) {
                document.getElementById('form-error').textContent = "Please fill out Item Name (or Scan), Store, and Price correctly.";
                return;
            }
            
            let regionCode = null;
            if (zipCode.trim() && /^\d{5}$/.test(zipCode.trim())) {
                regionCode = zipCode.trim().charAt(0);
            } else if (zipCode.trim()) {
                 document.getElementById('form-error').textContent = "Warning: Invalid zip code entered. Saving without region filter.";
            }

            document.getElementById('form-error').textContent = ""; 

            const logBtn = document.getElementById('log-price-btn');
            logBtn.disabled = true;
            logBtn.textContent = 'Saving...';

            try {
                // Keep the itemName as is if it's a barcode string, otherwise normalize it.
                let normalizedItemName = itemName.trim();
                const isScannedEntry = normalizedItemName.startsWith('[UPC:'); // NEW FIELD LOGIC
                
                if (!isScannedEntry) {
                    normalizedItemName = normalizedItemName.charAt(0).toUpperCase() + normalizedItemName.slice(1).toLowerCase();
                }

                const normalizedStoreName = store.trim().charAt(0).toUpperCase() + store.trim().slice(1).toLowerCase();

                await addDoc(collection(window.db, PUBLIC_PRICES_COLLECTION), {
                    itemName: normalizedItemName,
                    store: normalizedStoreName,
                    price: parseFloat(price.toFixed(2)),
                    timestamp: serverTimestamp(),
                    userId: window.userId,
                    regionCode: regionCode,
                    isScannedPrice: isScannedEntry // NEW FIELD
                });

                document.getElementById('log-form').reset();
                document.getElementById('form-error').textContent = `Success! Logged ${normalizedItemName}. ${regionCode ? `(Region ${regionCode})` : ''}`;
                setTimeout(() => document.getElementById('form-error').textContent = "", 3000);

            } catch (e) {
                console.error("Error adding document: ", e);
                document.getElementById('form-error').textContent = `An error occurred while saving: ${e.message}`;
            } finally {
                logBtn.disabled = false;
                logBtn.textContent = 'Log Price';
            }
        }

        /**
         * Deletes all log entries associated with a specific item name.
         * Only allows deletion if the current user owns at least one of the logs for that item.
         */
        window.deleteItemLogs = async function(itemName) {
            if (!window.db || !window.userId) {
                console.error("Database not ready or user not authenticated.");
                return;
            }

            const isConfirmed = await window.showCustomModal(
                `Delete '${itemName}'?`,
                `Are you sure you want to delete ALL price logs for "${itemName}"? This action cannot be undone.`,
                'Confirm Delete',
                'bg-red-600'
            );

            if (!isConfirmed) return;

            // Query all documents for this specific item name
            const q = query(collection(window.db, PUBLIC_PRICES_COLLECTION), where("itemName", "==", itemName));
            
            let logsToDelete = [];
            let isUserOwner = false;

            try {
                const snapshot = await getDocs(q);

                snapshot.forEach(doc => {
                    const data = doc.data();
                    if (data.userId === window.userId) {
                        isUserOwner = true; // Flag that this item was logged by the current user
                    }
                    logsToDelete.push({ docRef: doc.ref, data: data });
                });

                // Client-side simulation of the security rule: only proceed if the user is an owner
                if (!isUserOwner) {
                    await window.showCustomModal("Permission Denied", "You can only delete items that you initially added (logged by your User ID).", 'OK', 'bg-gray-500', false);
                    return;
                }
                
                // Delete the logs
                let deleteCount = 0;
                const logBtn = document.getElementById('log-price-btn');
                logBtn.disabled = true;

                for (const log of logsToDelete) {
                    await deleteDoc(log.docRef);
                    deleteCount++;
                }
                
                // The onSnapshot listener will handle the UI refresh
                document.getElementById('form-error').textContent = `Successfully deleted ${deleteCount} logs for '${itemName}'.`;
                setTimeout(() => document.getElementById('form-error').textContent = "", 3000);

            } catch (e) {
                console.error("Error deleting item logs: ", e);
                document.getElementById('form-error').textContent = `An error occurred while deleting: ${e.message}`;
            } finally {
                const logBtn = document.getElementById('log-price-btn');
                if (logBtn) logBtn.disabled = false; // Re-enable if it exists
            }
        }
        
        // --- Regional Filter Logic (UNCHANGED) ---

        /**
         * Applies the regional filter based on the entered zip code.
         */
        window.applyRegionFilter = function() {
            const zipCode = document.getElementById('filter-zip-code').value.trim();
            const filterErrorEl = document.getElementById('filter-error');
            const filterDisplayEl = document.getElementById('active-filter-display');

            if (!/^\d{5}$/.test(zipCode)) {
                filterErrorEl.textContent = "Please enter a valid 5-digit US zip code.";
                return;
            }

            filterErrorEl.textContent = "";
            window.activeRegionCode = zipCode.charAt(0);
            
            filterDisplayEl.innerHTML = `<span class="font-bold text-indigo-700">Active Filter:</span> Prices in Region ${window.activeRegionCode} (Based on ${zipCode})`;
            filterDisplayEl.classList.remove('text-gray-500');
            filterDisplayEl.classList.add('text-indigo-700');
            
            document.getElementById('clear-region-btn').classList.remove('hidden');
            document.getElementById('apply-region-btn').classList.add('hidden');

            window.renderComparisonList();
            window.renderShopCartSummary();
        }

        /**
         * Clears the regional filter.
         */
        window.clearRegionFilter = function() {
            window.activeRegionCode = null;
            document.getElementById('filter-zip-code').value = '';
            
            document.getElementById('active-filter-display').textContent = 'No Regional Filter Active. Showing All Logged Prices.';
            document.getElementById('active-filter-display').classList.add('text-gray-500');
            document.getElementById('active-filter-display').classList.remove('text-indigo-700');

            document.getElementById('clear-region-btn').classList.add('hidden');
            document.getElementById('apply-region-btn').classList.remove('hidden');
            
            window.renderComparisonList();
            window.renderShopCartSummary();
        }


        // --- Comparison List Rendering (MODIFIED for Delete Button) ---

        /**
         * Renders the comparison dashboard based on the aggregated data, active filters, and search term.
         */
        window.renderComparisonList = function() {
            const comparisonList = document.getElementById('comparison-list');
            comparisonList.innerHTML = '';
            
            // 1. Filter items by search term
            let filteredItems = Object.keys(window.allPricesMap).filter(itemName => 
                itemName.toLowerCase().includes(window.searchTerm)
            ).sort();

            const itemsToRender = [];
            const newStoreFilters = new Set(); 

            // 2. Further filter items based on the active region code
            filteredItems.forEach(itemName => {
                const itemData = window.allPricesMap[itemName];
                const entries = itemData.allEntries.filter(entry => 
                    !window.activeRegionCode || entry.regionCode === window.activeRegionCode
                );

                if (entries.length > 0) {
                    // Re-aggregate best/worst prices based *only* on filtered entries
                    const reAggregatedData = aggregateItemData(entries, itemName);
                    itemsToRender.push(reAggregatedData);

                    // Update available stores based on the filtered data
                    Object.keys(reAggregatedData.entriesByStore).forEach(store => newStoreFilters.add(store));
                }
            });

            // Update unique stores list and re-render the store filters based on available stores
            window.uniqueStores = Array.from(newStoreFilters).sort();
            updateStoreFilters();


            if (itemsToRender.length === 0) {
                const emptyMessage = window.activeRegionCode
                    ? `No prices logged yet for Region ${window.activeRegionCode}. Try logging one!`
                    : (window.searchTerm ? `No items found matching "${window.searchTerm}".` : 'No prices logged yet. Be the first!');
                
                comparisonList.innerHTML = `<p class="text-gray-500 p-4">${emptyMessage}</p>`;
                return;
            }
            
            const activeStores = Array.from(window.activeStoreFilters).filter(store => window.uniqueStores.includes(store)); 
            const numCols = Math.min(activeStores.length + 1, 5);
            const gridColsClass = `grid-cols-1 sm:grid-cols-${numCols}`;

            itemsToRender.forEach(itemData => {
                const { itemName, bestPrice, bestStore, worstPrice, worstStore, entriesByStore, isOwnedByCurrentUser } = itemData; // Added isOwnedByCurrentUser
                const itemIcon = getItemIcon(itemName); 
                const isInCart = window.shopCart.has(itemName); 

                const itemElement = document.createElement('div');
                itemElement.className = 'bg-white p-4 mb-3 rounded-xl shadow-lg border-t-4 border-indigo-300';
                
                // --- Item Header (Name and Best Price) ---
                const headerHtml = `
                    <div class="flex justify-between items-center mb-3 border-b pb-2">
                        <div class="flex items-center space-x-3">
                            <h3 class="text-lg font-bold text-gray-800 flex items-center space-x-2">
                                <span class="text-2xl">${itemIcon}</span>
                                <span>${itemName}</span>
                            </h3>
                            <button onclick="window.toggleCart('${itemName.replace(/'/g, "\\'")}')" 
                                    class="p-1 rounded-full transition duration-150 ${isInCart ? 'bg-red-500 text-white hover:bg-red-600' : 'bg-green-100 text-green-700 hover:bg-green-200'}"
                                    title="${isInCart ? 'Remove from Cart' : 'Add to Cart'}">
                                🛒
                            </button>
                            <!-- NEW: Delete button visible only if the item was logged by the current user -->
                            ${isOwnedByCurrentUser ? `
                                <button onclick="window.deleteItemLogs('${itemName.replace(/'/g, "\\'")}')" 
                                        class="p-1 rounded-full transition duration-150 bg-red-100 text-red-700 hover:bg-red-200"
                                        title="Delete All Logs for this Item (Requires Ownership)">
                                    <svg class="h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg>
                                </button>
                            ` : ''}
                        </div>
                        <div class="text-right">
                            <p class="text-xs font-semibold uppercase text-emerald-600">Cheapest Price</p>
                            <span class="text-2xl font-extrabold text-emerald-600">$${bestPrice.toFixed(2)}</span>
                            <p class="text-xs text-gray-500">at ${bestStore}</p>
                        </div>
                    </div>
                `;

                // --- Store Comparison Grid ---
                let comparisonGridHtml = '';
                
                if (activeStores.length > 0) {
                    comparisonGridHtml += `<div class="grid gap-3 ${gridColsClass} text-center">`;
                    
                    activeStores.forEach(store => {
                        const storeEntry = entriesByStore[store];
                        const storePrice = storeEntry ? `$${storeEntry.latestPrice.toFixed(2)}` : 'N/A';
                        
                        let priceClass = 'text-gray-700 bg-gray-50';
                        if (storeEntry && storeEntry.latestPrice === bestPrice) {
                            priceClass = 'text-emerald-600 font-bold bg-emerald-50';
                        } else if (storeEntry && storeEntry.latestPrice === worstPrice) {
                            priceClass = 'text-red-600 font-bold bg-red-50';
                        }
                        
                        comparisonGridHtml += `
                            <div class="p-2 rounded-lg shadow-inner ${priceClass.split(' ').filter(c => !c.startsWith('bg-')).join(' ')}">
                                <p class="text-xs font-semibold mb-1 text-indigo-700">${store}</p>
                                <span class="text-lg ${priceClass.split(' ').filter(c => !c.startsWith('text-')).join(' ')} ${priceClass.split(' ').filter(c => c.startsWith('text-')).join(' ')}">${storePrice}</span>
                                <p class="text-xs text-gray-400 mt-1">${storeEntry ? storeEntry.latestDate.toLocaleDateString() : ''}</p>
                            </div>
                        `;
                    });
                    comparisonGridHtml += `</div>`;
                } else {
                    comparisonGridHtml = `<p class="text-sm text-gray-600 mt-2">Worst Price Logged: <span class="font-bold text-red-600">$${worstPrice.toFixed(2)}</span> at ${worstStore}. Select stores above to compare side-by-side.</p>`;
                }

                itemElement.innerHTML = headerHtml + comparisonGridHtml;
                comparisonList.appendChild(itemElement);
            });
        }


        /**
         * Helper function to aggregate data for a single item from a set of entries.
         */
        function aggregateItemData(entries, itemName) {
            let bestPrice = Infinity;
            let bestStore = '';
            let worstPrice = -Infinity;
            let worstStore = '';
            const entriesByStore = {}; 
            let isOwnedByCurrentUser = false; // NEW: Flag to check for delete permission

            entries.forEach(data => {
                const price = data.price;
                const store = data.store;
                const date = data.timestamp instanceof Date ? data.timestamp : (data.timestamp ? data.timestamp.toDate() : new Date());

                if (price < bestPrice) {
                    bestPrice = price;
                    bestStore = store;
                }
                if (price > worstPrice) {
                    worstPrice = price;
                    worstStore = store;
                }
                
                // NEW: Check for item ownership
                if (data.userId === window.userId) {
                    isOwnedByCurrentUser = true;
                }

                // Track the latest price for this specific store/item combination
                if (!entriesByStore[store] || date > entriesByStore[store].latestDate) {
                    entriesByStore[store] = {
                        latestPrice: price,
                        latestDate: date
                    };
                }
            });

            return {
                itemName,
                bestPrice,
                bestStore,
                worstPrice,
                worstStore,
                entriesByStore,
                isOwnedByCurrentUser // Included
            };
        }

        /**
         * Listens for real-time price updates, aggregates data, and triggers UI updates.
         */
        window.loadPrices = function() {
            if (!window.db) return;

            const q = query(collection(window.db, PUBLIC_PRICES_COLLECTION));

            onSnapshot(q, (querySnapshot) => {
                window.allPricesMap = {};
                const newStoreSet = new Set();
                
                // Re-run the seeding check every time data loads
                const seedButtonContainer = document.getElementById('seed-button-container');
                if (querySnapshot.empty) {
                    seedButtonContainer.classList.remove('hidden');
                } else {
                    seedButtonContainer.classList.add('hidden');
                }


                querySnapshot.forEach((doc) => {
                    const data = doc.data();
                    const itemName = data.itemName;
                    const store = data.store;
                    const price = data.price;
                    const date = data.timestamp instanceof Date ? data.timestamp : (data.timestamp ? data.timestamp.toDate() : new Date());
                    
                    // 1. Track all unique stores (globally, before filtering)
                    newStoreSet.add(store);

                    // 2. Aggregate raw data by ItemName (Keep all entries for client-side region filtering)
                    if (!window.allPricesMap[itemName]) {
                        window.allPricesMap[itemName] = {
                            allEntries: []
                        };
                    } 
                    
                    // Add the raw entry to the list
                    window.allPricesMap[itemName].allEntries.push({
                        price: price,
                        store: store,
                        timestamp: date,
                        regionCode: data.regionCode, // Store region code for filtering
                        userId: data.userId // Store userId for ownership check
                    });
                });
                
                // Post-processing: Compute the aggregated data structures
                Object.keys(window.allPricesMap).forEach(itemName => {
                    const itemData = window.allPricesMap[itemName];
                    // Aggregate all data to initialize the object structure
                    Object.assign(itemData, aggregateItemData(itemData.allEntries, itemName));
                });


                // Update the global unique stores list
                window.uniqueStores = Array.from(newStoreSet).sort();
                
                // Re-render both list and cart summary
                window.renderComparisonList();
                window.renderShopCartSummary(); 

            }, (error) => {
                console.error("Firestore snapshot error: ", error);
                document.getElementById('comparison-list').innerHTML = `<p class="text-red-500 p-4">Error loading data: ${error.message}</p>`;
            });
        }
        
        // --- Filter Logic (UNCHANGED) ---

        window.handleSearchInput = function(term) {
            window.searchTerm = term.toLowerCase().trim();
            window.renderComparisonList();
        }

        window.handleStoreFilterChange = function(storeName, isChecked) {
            if (isChecked) {
                window.activeStoreFilters.add(storeName);
            } else {
                window.activeStoreFilters.delete(storeName);
            }
            window.renderComparisonList();
        }
        
        // Updates the checkboxes based on available stores in the currently filtered data
        function updateStoreFilters() {
            const filterContainer = document.getElementById('store-filter-container');
            filterContainer.innerHTML = '';
            
            if (window.uniqueStores.length === 0) {
                 filterContainer.innerHTML = '<p class="text-xs text-gray-500 p-2">No store data in this view.</p>';
                 return;
            }

            window.uniqueStores.forEach(store => {
                const isChecked = window.activeStoreFilters.has(store);
                const filterId = `filter-${store.replace(/\s/g, '-')}`;

                const div = document.createElement('div');
                div.className = 'flex items-center space-x-2';
                div.innerHTML = `
                    <input type="checkbox" id="${filterId}" value="${store}" ${isChecked ? 'checked' : ''} 
                           class="h-4 w-4 text-indigo-600 border-gray-300 rounded focus:ring-indigo-500 focus:border-indigo-500"
                           onchange="window.handleStoreFilterChange('${store}', this.checked)">
                    <label for="${filterId}" class="text-sm font-medium text-gray-700 select-none">${store}</label>
                `;
                filterContainer.appendChild(div);
            });
        }
        


        // Start the application initialization when the window loads
        window.onload = initializeAndAuthenticate;
    </script>
</head>
<body class="min-h-screen">
    
    <!-- 1. Menu Button (Fixed Top Left) -->
    <button onclick="window.toggleSidebar()" class="fixed z-50 top-4 left-4 text-gray-800 bg-white p-2 rounded-full shadow-lg hover:bg-indigo-50 transition lg:hidden" aria-label="Toggle Menu">
        <!-- Hamburger Icon -->
        <svg class="h-6 w-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"></path>
        </svg>
    </button>

    <!-- 2. Sidebar Menu -->
    <nav id="sidebar" class="fixed top-0 left-0 w-64 h-full bg-gray-900 text-white z-40 transform -translate-x-full transition-transform duration-300 ease-in-out shadow-2xl">
        <div class="p-6">
            <h3 class="text-3xl font-bold text-indigo-400 mb-8">App Menu</h3>
            <ul class="space-y-4">
                <li>
                    <a href="#" onclick="window.handleMenuAction('Dashboard')" class="flex items-center p-3 rounded-lg hover:bg-indigo-700 transition">
                        <svg class="h-5 w-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0h6m-6 0v-4a1 1 0 011-1h2a1 1 0 011 1v4"></path></svg>
                        <span>Dashboard (Home)</span>
                    </a>
                </li>
                <li>
                    <a href="#" onclick="window.handleMenuAction('App Settings')" class="flex items-center p-3 rounded-lg hover:bg-indigo-700 transition">
                        <svg class="h-5 w-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
                        <span>App Settings</span>
                    </a>
                </li>
                <li>
                    <a href="#" onclick="window.handleMenuAction('App Guide')" class="flex items-center p-3 rounded-lg hover:bg-indigo-700 transition">
                        <svg class="h-5 w-5 mr-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2m-7 6h4"></path></svg>
                        <span>App Guide (Plan)</span>
                    </a>
                </li>
            </ul>
        </div>
    </nav>

    <!-- 3. Overlay -->
    <div id="sidebar-backdrop" onclick="window.toggleSidebar()" class="fixed inset-0 bg-black opacity-0 z-30 transition-opacity duration-300 pointer-events-none"></div>

    <!-- Main Content Container -->
    <div class="container mx-auto p-4 md:p-8">

        <!-- Header -->
        <header class="text-center mb-8">
            <h1 class="text-4xl font-extrabold text-gray-900 mb-2">🛒 Regional Grocery Price Compare</h1>
            <p class="text-xl text-gray-600">Build your cart and instantly compare localized totals across stores.</p>
            <p id="user-id-display" class="text-xs text-gray-400 mt-2"></p>
        </header>

        <!-- Status Message -->
        <div id="status-message" class="text-center text-sm font-medium text-indigo-600 mb-6">Initializing App...</div>

        <!-- Layout: Log Form + Cart Summary (Left) and Comparison Dashboard (Right) -->
        <div class="grid grid-cols-1 lg:grid-cols-4 gap-8">

            <!-- Price Logging Form & Shop Cart Summary (Left Column) -->
            <div class="lg:col-span-1 space-y-8">
                
                <!-- 1. Price Logging Form -->
                <div class="bg-white p-6 rounded-2xl shadow-xl h-fit border-t-4 border-indigo-600">
                    <h2 class="text-2xl font-semibold mb-4 text-gray-800 border-b pb-2">Log a New Price</h2>
                    
                    <div class="mb-4">
                        <button onclick="window.openScannerModal()" class="w-full flex items-center justify-center space-x-2 bg-emerald-500 text-white p-3 rounded-lg font-bold hover:bg-emerald-600 transition duration-150 shadow-md">
                            <svg class="h-6 w-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 7v10M10 7v10M16 7v10M20 7v10M6 5h12a2 2 0 012 2v10a2 2 0 01-2 2H6a2 2 0 01-2-2V7a2 2 0 012-2z"></path></svg>
                            <span>Scan Price (Recommended)</span>
                        </button>
                    </div>

                    <form id="log-form" onsubmit="event.preventDefault(); window.savePrice();">
                        <div class="mb-4">
                            <label for="item-name" class="block text-sm font-medium text-gray-700 mb-1">Item Name / Barcode ID</label>
                            <input type="text" id="item-name" required class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" placeholder="e.g., Gala Apples or [UPC: 1234...]" />
                            <p class="text-xs text-gray-500 mt-1">Manual entry is fine, but scanning provides a unique item ID.</p>
                        </div>

                        <div class="mb-4">
                            <label for="store-name" class="block text-sm font-medium text-gray-700 mb-1">Store Name</label>
                            <input type="text" id="store-name" required class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" placeholder="e.g., Walmart, Kroger" />
                        </div>

                        <div class="mb-4">
                            <label for="item-price" class="block text-sm font-medium text-gray-700 mb-1">Price ($)</label>
                            <input type="number" step="0.01" min="0.01" id="item-price" required class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" placeholder="e.g., 4.59" />
                        </div>

                        <!-- Zip Code Input for Logging -->
                        <div class="mb-6">
                            <label for="log-zip-code" class="block text-sm font-medium text-gray-700 mb-1">Your Zip Code (Optional, for Region)</label>
                            <input type="text" id="log-zip-code" maxlength="5" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" placeholder="e.g., 90210" />
                        </div>

                        <button type="submit" id="log-price-btn" class="w-full bg-indigo-600 text-white p-3 rounded-lg font-bold hover:bg-indigo-700 transition duration-150 shadow-md disabled:bg-indigo-400">
                            Log Price
                        </button>
                        <p id="form-error" class="mt-3 text-sm text-center text-red-600 font-medium"></p>
                    </form>

                    <!-- Database Seeding Option -->
                    <div id="seed-button-container" class="mt-6 pt-4 border-t border-gray-200 hidden">
                        <button onclick="window.showSeedModal()" class="w-full bg-emerald-500 text-white p-3 rounded-lg font-bold hover:bg-emerald-600 transition duration-150 shadow-md text-sm">
                            Seed Database with Sample Items
                        </button>
                    </div>
                </div>

                <!-- 2. Regional Filter Input -->
                <div class="bg-indigo-50 p-6 rounded-2xl shadow-xl h-fit border border-indigo-200">
                    <h2 class="text-xl font-semibold mb-4 text-indigo-800 border-b pb-2">Regional Price Filter</h2>
                    
                    <div class="mb-4">
                        <label for="filter-zip-code" class="block text-sm font-medium text-indigo-700 mb-1">Enter Zip Code to Filter Prices:</label>
                        <input type="text" id="filter-zip-code" maxlength="5" class="w-full p-3 border border-indigo-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" placeholder="e.g., 78701" />
                    </div>

                    <button id="apply-region-btn" onclick="window.applyRegionFilter()" class="w-full bg-indigo-500 text-white p-3 rounded-lg font-bold hover:bg-indigo-600 transition duration-150 shadow-md disabled:bg-indigo-300">
                        Apply Region Filter
                    </button>
                    
                    <button id="clear-region-btn" onclick="window.clearRegionFilter()" class="w-full bg-gray-400 text-white p-3 rounded-lg font-bold hover:bg-gray-500 transition duration-150 shadow-md mt-2 hidden">
                        Clear Region Filter
                    </button>

                    <p id="filter-error" class="mt-3 text-sm text-center text-red-600 font-medium"></p>
                    <p id="active-filter-display" class="mt-3 text-xs text-center text-gray-500">
                        No Regional Filter Active. Showing All Logged Prices.
                    </p>
                </div>

                <!-- 3. Shop Cart Summary -->
                <div class="bg-white p-6 rounded-2xl shadow-xl h-fit">
                    <h2 class="text-2xl font-semibold mb-4 text-gray-800 border-b pb-2 flex items-center justify-between">
                        Shop Cart Total 
                        <span id="cart-item-count" class="text-base font-normal text-gray-500">
                             (0 items)
                        </span>
                    </h2>
                    
                    <div class="mb-4">
                        <p class="text-sm font-medium text-gray-600 mb-2">Items in Cart:</p>
                        <ul id="cart-item-list" class="divide-y divide-gray-100 max-h-40 overflow-y-auto border rounded-lg bg-gray-50">
                            <!-- Cart items rendered here -->
                        </ul>
                    </div>

                    <p class="text-sm font-medium text-gray-600 mb-2 mt-4 border-t pt-2">Cart Cost By Store:</p>
                    <div id="cart-summary-totals" class="p-1 rounded-lg space-y-1">
                        <!-- Cart totals rendered here -->
                    </div>
                </div>
            </div>

            <!-- Comparison Dashboard (Right Column) -->
            <div class="lg:col-span-3">
                <h2 class="text-2xl font-semibold mb-4 text-gray-800 border-b pb-2">Best Price Comparison</h2>
                
                <!-- Search Input -->
                <div class="bg-white p-4 rounded-xl shadow mb-4">
                    <p class="text-sm font-semibold text-gray-700 mb-2">Filter Items:</p>
                    <input type="text" id="item-search" 
                           oninput="window.handleSearchInput(this.value)" 
                           class="w-full p-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500" 
                           placeholder="Type item name or UPC to filter the list..." />
                </div>

                <!-- Store Filter Controls -->
                <div class="bg-indigo-100 p-4 rounded-xl shadow-inner mb-6">
                    <p class="text-sm font-semibold text-indigo-800 mb-2">Select Stores to Compare Side-by-Side:</p>
                    <div id="store-filter-container" class="flex flex-wrap gap-x-4 gap-y-2">
                        <!-- Filters injected here -->
                        <p class="text-xs text-gray-500 p-2">Loading stores...</p>
                    </div>
                </div>

                <div id="comparison-list" class="space-y-4">
                    <!-- Price comparison items will be injected here by JavaScript -->
                    <div class="flex justify-center items-center h-24 bg-gray-200 rounded-xl">
                        <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-gray-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                        </svg>
                        <span class="text-gray-600">Loading price data...</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <!-- Seeding Confirmation Modal -->
    <div id="seed-modal" class="modal fixed inset-0 z-50 flex items-center justify-center bg-gray-900 bg-opacity-75">
        <div class="bg-white p-8 rounded-xl shadow-2xl max-w-lg w-full m-4 transform transition-all">
            <h3 class="text-xl font-bold mb-4 text-gray-900">Confirm Database Seeding</h3>
            <p id="seed-status" class="text-gray-700 mb-6">Ready to load 12 sample price entries into the database. This will help demonstrate the regional filtering feature.</p>
            <div class="flex justify-end space-x-4">
                <button onclick="window.closeSeedModal()" class="px-6 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 transition">
                    Cancel
                </button>
                <button id="seed-confirm-btn" onclick="window.seedDatabase()" class="px-6 py-2 bg-emerald-500 text-white rounded-lg font-semibold hover:bg-emerald-600 transition">
                    Proceed and Seed
                </button>
            </div>
        </div>
    </div>
    
    <!-- Barcode Scanner Modal (NEW) -->
    <div id="scanner-modal" class="modal fixed inset-0 z-50 flex flex-col items-center justify-center bg-gray-900 bg-opacity-90 p-4">
        <div class="relative w-full max-w-xl mx-auto p-4 rounded-xl">
            <h3 class="text-xl font-bold mb-4 text-white text-center">Scan Product Barcode</h3>
            
            <div class="relative bg-black rounded-xl shadow-2xl overflow-hidden">
                <video id="scanner-video" playsinline></video>
                <div id="scan-box"></div> 
            </div>
            
            <p id="scanner-feedback" class="text-center text-sm font-medium text-yellow-300 mt-4"></p>
            
            <button onclick="window.closeScannerModal()" class="mt-6 w-full bg-red-600 text-white p-3 rounded-lg font-bold hover:bg-red-700 transition duration-150 shadow-lg">
                Cancel Scan
            </button>
        </div>
    </div>

    <!-- Generic Confirmation Modal (Replaces alert/confirm) -->
    <div id="custom-confirm-modal" class="modal fixed inset-0 z-50 flex items-center justify-center bg-gray-900 bg-opacity-75">
        <div class="bg-white p-8 rounded-xl shadow-2xl max-w-sm w-full m-4 transform transition-all">
            <h3 id="custom-modal-title" class="text-xl font-bold mb-4 text-gray-900"></h3>
            <p id="custom-modal-message" class="text-gray-700 mb-6"></p>
            <div class="flex justify-end space-x-4">
                <button id="custom-modal-cancel" class="px-6 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 transition hidden">
                    Cancel
                </button>
                <button id="custom-modal-confirm" class="px-6 py-2 text-white rounded-lg font-semibold transition">
                    OK
                </button>
            </div>
        </div>
    </div>
</body>
</html>
