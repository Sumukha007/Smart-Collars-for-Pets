<!DOCTYPE html>
<html>
<head>
    <title>Smart Collars for Pets</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
    <style>
        body, html {
            height: 100%;
            margin: 0;
            font-family: Arial, Helvetica, sans-serif;
        }
        .bg {
            background-image: url('bg.avif'); /* Add your background image */
            height: 100%;
            background-position: center;
            background-repeat: no-repeat;
            background-size: cover;
            position: relative;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            flex-direction: column;
        }
        #map {
            height: 50vh;
            width: 60vw;
            border: 2px solid #fff;
            border-radius: 10px;
            display: none; /* Initially hidden */
        }
        .container {
            text-align: center;
            padding: 20px;
            background-color: rgba(255, 255, 255, 0.8);
            border-radius: 10px;
        }
        h1 {
            font-size: 2.5em;
            margin-bottom: 20px;
            color: black; /* Change heading color to black */
        }
        p {
            color: black; /* Change "Select your pet" color to black */
        }
        .input-field {
            margin: 10px;
            padding: 10px;
            font-size: 1em;
            width: 250px; /* Adjusted width for full display */
        }
        .btn {
            padding: 10px 20px;
            font-size: 1em;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin: 5px;
        }
        .btn:hover {
            background-color: #0056b3;
        }
        .warning {
            color: red;
            font-size: 1.2em;
            margin-top: 10px;
        }
    </style>
</head>
<body>
    <div class="bg">
        <div class="container">
            <h1>Smart Collars for Pets</h1>
            <select id="petType" class="input-field" onchange="handlePetTypeChange()">
                <option value="">Select Pet</option>
                <option value="dog">Dog</option>
                <option value="cat">Cat</option>
            </select>
            <input type="text" id="collarId" class="input-field" placeholder="Enter collar ID" maxlength="8" onchange="resetLocation()">
            <input type="number" id="radius" class="input-field" placeholder="Enter boundary radius in meters" min="0">
            <button class="btn" onclick="initializeMap()">Start Tracking</button>
            <button class="btn" onclick="stopTracking()">Stop Tracking</button>
            <div id="map"></div>
            <div id="warning" class="warning"></div>
            <button class="btn" onclick="showLocationHistory()">Show Location History</button>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
    <script>
        var campusCenter = [13.133495151187617, 77.56798499887222]; // BMS Institute of Technology and Management coordinates (lat, lng)
        var locationHistory = [];
        var maxHistory = 5;
        var map;
        var marker;
        var circle;
        var currentLocation = campusCenter;
        var tracking = false;

        function handlePetTypeChange() {
            stopTracking();
            updateCollarIdPrefix();
            resetLocation();
            document.getElementById('map').style.display = 'none'; // Hide the map until tracking is restarted
        }

        function updateCollarIdPrefix() {
            var petType = document.getElementById('petType').value;
            var collarIdInput = document.getElementById('collarId');
            collarIdInput.value = petType ? petType.substring(0, 3) : '';
            collarIdInput.setSelectionRange(collarIdInput.value.length, collarIdInput.value.length);
        }

        function resetLocation() {
            currentLocation = campusCenter;
            locationHistory = [];
            document.getElementById('warning').textContent = '';

            if (marker) {
                marker.setLatLng(campusCenter);
                marker.bindPopup('BMS Institute of Technology and Management').openPopup();
            }
            if (circle) {
                circle.setLatLng(campusCenter);
            }
            if (map) {
                map.setView(campusCenter, 15);
            }
        }

        function initializeMap() {
            var petType = document.getElementById('petType').value;
            var radius = document.getElementById('radius').value;
            var collarId = document.getElementById('collarId').value;

            if (!petType) {
                alert('Please select a pet type.');
                return;
            }
            if (!radius) {
                alert('Please enter a boundary radius.');
                return;
            }
            if (!validateCollarId(collarId, petType)) {
                alert('Invalid collar ID. It should start with "' + petType + '" and be 8 characters long.');
                return;
            }

            document.getElementById('map').style.display = 'block'; // Show the map
            
            if (!map) {
                map = L.map('map').setView(campusCenter, 15);

                L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
                }).addTo(map);

                marker = L.marker(campusCenter).addTo(map)
                    .bindPopup('BMS Institute of Technology and Management')
                    .openPopup();

                circle = L.circle(campusCenter, {
                    color: 'red',
                    fillColor: '#f03',
                    fillOpacity: 0.2,
                    radius: radius
                }).addTo(map);
            } else {
                circle.setRadius(radius);
            }

            tracking = true;
            updateLocation();
        }

        function validateCollarId(collarId, petType) {
            var prefix = petType === "dog" ? "dog" : "cat";
            var regex = new RegExp("^" + prefix + "\\d{5}$");
            return regex.test(collarId);
        }

        function updateLocation() {
            if (!tracking) return;

            var delta = 0.0001; // Approx. 10 meters
            var newLat = currentLocation[0] + (Math.random() - 0.5) * delta;
            var newLng = currentLocation[1] + (Math.random() - 0.5) * delta;
            var newPosition = [newLat, newLng];

            locationHistory.push(newPosition);
            if (locationHistory.length > maxHistory) {
                locationHistory.shift(); // Remove oldest location
            }

            currentLocation = newPosition;
            map.setView(newPosition);

            marker.setLatLng(newPosition);
            marker.bindPopup('Current Location').openPopup();

            var radius = document.getElementById('radius').value;
            circle.setLatLng(campusCenter).setRadius(radius);

            var distance = calculateDistance(newPosition, campusCenter);
            if (distance > radius) {
                alert('Warning: Pet has left the boundary!');
                document.getElementById('warning').textContent = 'Warning: Pet has left the boundary!';
            } else {
                document.getElementById('warning').textContent = '';
            }

            setTimeout(updateLocation, 3000); // Continue updating location
        }

        function calculateDistance(coord1, coord2) {
            var R = 6371e3; // Earth radius in meters
            var lat1 = coord1[0] * Math.PI / 180;
            var lat2 = coord2[0] * Math.PI / 180;
            var deltaLat = (coord2[0] - coord1[0]) * Math.PI / 180;
            var deltaLng = (coord2[1] - coord1[1]) * Math.PI / 180;

            var a = Math.sin(deltaLat / 2) * Math.sin(deltaLat / 2) +
                    Math.cos(lat1) * Math.cos(lat2) *
                    Math.sin(deltaLng / 2) * Math.sin(deltaLng / 2);
            var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

            return R * c; // Distance in meters
        }

        function stopTracking() {
            tracking = false;
        }

        function showLocationHistory() {
            var popup = window.open("", "LocationHistory", "width=400,height=600");
            popup.document.write("<h3>Location History</h3>");
            locationHistory.forEach((loc, index) => {
                popup.document.write(`<p>Location ${index + 1}: [${loc[0].toFixed(6)}, ${loc[1].toFixed(6)}]</p>`);
            });
        }
    </script>
</body
