# Basecamp

Web portal prototype for Basecamp, the adventure travel app with the Scout agent.

- `/` is the traveler portal: Nearby (opens on your current location), Plan with Scout, Trips, Rides, Tickets & documents, Safety.
- `/platform` is the platform console: operator dashboard, groups and payments, listings, and commission settlement (all Basecamp commission goes to eSewa 9768786270).

Static HTML, CSS and JavaScript with no build step. Deploys to Vercel as a static site (`vercel.json`).

Status: prototype. Bookings, trips and payments are sample data, and no money moves. Live Scout suggestions only work inside the Claude artifact viewer; on other hosts Nearby falls back to sample places until a backend with real data sources is added.
