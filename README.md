# ONE DECK Riftbound Collection Tracker

**Status:** Personal-use application / private development  
**Scope:** Riftbound only

### 1. Overview

This is a personal-use collection tracking application for **Riftbound**. There are currently **no plans for a public release**.

The application is designed around tracking the user's **physical card collection**, rather than simulating or reproducing the game itself. Every physical card is tracked as an individual copy, with information such as its printing, condition, acquisition, value, and physical storage location.

The application is intended to function primarily as an **offline card catalog and collection manager**.

### 2. Primary Use Cases

The application allows the user to:

- Browse, search, and filter the Riftbound card catalog.
- Record individual physical card copies as they are acquired.
- Track where each physical card is stored, such as in a binder, storage box, or deck box.
- Track acquisition information such as date, source, and cost.
- Track the current collection value.
- Organize cards into physical containers.
- Track trades, lending, and borrowing.
- Track physical decks and the cards currently assigned to them.
- Maintain decklists and compare them with the cards physically owned.

The emphasis is on **collection management and physical organization**, not gameplay.

### 3. Deck Management

> **Deck management, not deckbuilding.**

The application may store decklists and physical decks so the user can keep track of which cards are currently assigned to a deck and which cards are missing.

It is **not a deckbuilding or gameplay tool**. It does not simulate Riftbound gameplay or provide gameplay functionality.

The MVP does not include a full deckbuilder with card discovery, synergy recommendations, metagame statistics, or similar functionality.

### 4. Offline-First Catalog

The card catalog is designed to remain available **offline**.

Riftbound card data and images are imported into a local catalog and distributed to the application as versioned catalog/image releases. User devices do not need to fetch card data or images from live third-party websites during normal use.

The application can therefore be used while handling and organizing a physical collection without requiring a live connection to an external card database.

### 5. Riftbound Catalog Source

The **Riot Games Riftbound API** is the authoritative source for Riftbound catalog data used by the application.

The application uses the Riot API specifically to obtain Riftbound content, including:

- Card names
- Set information
- Rarity
- Card type
- Domains / faction information
- Rules text
- Flavor text
- Keywords and tags
- Card statistics
- Collector numbers
- Card artwork and image URLs

The Riot-facing integration is based on the `riftbound-content-v1` endpoint described in Riot's developer portal.

The application does not use third-party or community card databases as the source of Riftbound card data or card images.

### 6. API Key Usage

The Riot API is accessed by an internal **Catalog Studio** used during catalog updates.

The workflow is:

1. Catalog Studio authenticates to the Riot API using a Riot API key.
2. It retrieves the current Riftbound catalog.
3. The data is normalized into the application's internal catalog format.
4. Changes are reviewed and confirmed.
5. A versioned catalog release is produced.
6. The resulting catalog is made available to the user's offline-first application.

The production application does not need to query the Riot API continuously during normal collection use.

The API is therefore being used specifically as the **Riftbound catalog source**, rather than as a gameplay service.

### 7. Card and Printing Model

The application distinguishes between an abstract card and an individual physical printing.

A **card** represents the underlying Riftbound card.

A **printing** represents a specific physical version, including its set, collector number, rarity, variant, language, and image.

A **card copy** represents one physical card owned by the user.

This allows the collection tracker to distinguish between multiple physical copies and different Riftbound printings while still treating them as part of the same catalog.

### 8. Physical Collection Tracking

The central data model is based on **individual physical copies**, not only quantities.

For each physical card, the application can retain:

- Exact printing
- Finish
- Condition
- Acquisition information
- Cost basis
- Collection value snapshot
- Physical storage location
- Deck assignment
- Ownership status
- Loan status

Cards can be grouped visually in the interface for convenience, but the underlying records remain individual physical copies.

### 9. Products and Acquisition

The application can also track physical Riftbound products such as:

- Booster packs
- Booster boxes
- Cases
- Preconstructed products
- Bundles
- User-defined lots

Products can be opened and their physical card contents recorded, preserving provenance between the product and the individual cards obtained from it.

### 10. Data and Images

Riftbound catalog data and images are imported through the Riot API integration and packaged for offline use.

The application does not scrape websites or rely on unofficial Riftbound image sources for its catalog.

Third-party marketplace data may be used separately for collection valuation and marketplace-related information, but it is not the source of Riftbound card content or imagery.

### 11. Non-Goals

The Riftbound application is not intended to:

- Simulate Riftbound gameplay.
- Replace the Riftbound game client.
- Provide a gameplay service.
- Function as a full deckbuilding or metagame analysis platform.
- Publish or operate a public community database.
- Create a public marketplace for user collections.

### 12. Development Status

The application is currently being developed as a **private, personal-use collection tracker**.

The immediate purpose of the Riot API integration is to obtain and maintain an accurate, versioned Riftbound catalog for that application.

