# Vending Machine API: Project Debrief for Interview

This document provides a comprehensive summary of the 6 major bugs identified and fixed in the Vending Machine API project. It is designed to serve as a clear and concise guide for discussing the improvements made during your technical interview.

---

## 1. Critical Bug Fixes

Here is a detailed breakdown of the bugs that were resolved, the code changes made, and why these fixes are important for the stability and performance of the application.

### Bug 1: Race Condition in Purchase Function

*   **The Problem**: The original code had a critical flaw where two users could purchase the last available item at the same time, leading to a negative inventory count. This is a classic "race condition" that can occur in concurrent systems.
*   **The Fix**: I made the purchase operation "atomic" by using a database-level lock (`with_for_update()`). This ensures that once a purchase process begins for an item, no other process can modify that item until the first one is complete.

    **Before (The Bug):**
    ```python
    # in app/services/purchase_service.py
    def purchase(db: Session, item_id: str, cash_inserted: int) -> dict:
        item = db.query(Item).filter(Item.id == item_id).first()
        time.sleep(0.05)  # <-- This sleep window makes the race condition likely
        if item.quantity <= 0:
            raise ValueError("out_of_stock")
        # ... (rest of the logic)
    ```

    **After (The Fix):**
    ```python
    # in app/services/purchase_service.py
    def purchase(db: Session, item_id: str, cash_inserted: int) -> dict:
        # with_for_update() locks the row until the transaction is committed.
        item = db.query(Item).with_for_update().filter(Item.id == item_id).first()
        if item.quantity <= 0:
            raise ValueError("out_of_stock")
        # ... (rest of the logic)
    ```
*   **Why This Matters**: This fix is crucial for data integrity. It guarantees that inventory levels are always accurate, preventing the business from overselling products and ensuring the system is reliable under real-world concurrent use.

### Bug 2: The "N+1 Query" Performance Bottleneck

*   **The Problem**: The endpoint to get a full view of all slots and their items (`/slots/full-view`) was extremely inefficient. It made one query to fetch all slots, and then *another query for each slot* to fetch its items.
*   **The Fix**: I optimized the query using SQLAlchemy's `joinedload`. This tells the database to fetch all the slots and their corresponding items together in a single, efficient `JOIN` operation.

    **Before (The Bug):**
    ```python
    # in app/services/slot_service.py
    def get_full_view(db: Session) -> list[SlotFullView]:
        slots = db.query(Slot).all() # <-- First query
        result = []
        for slot in slots:
            # slot.items loaded per slot (N+1) <-- N more queries happen here
            items = [ ... for item in slot.items ]
            result.append(...)
        return result
    ```

    **After (The Fix):**
    ```python
    # in app/services/slot_service.py
    from sqlalchemy.orm import joinedload

    def get_full_view(db: Session) -> list[SlotFullView]:
        # Use joinedload to fetch slots and their items in one query.
        slots = db.query(Slot).options(joinedload(Slot.items)).all()
        # ... (rest of the function is the same, but now it's efficient)
    ```
*   **Why This Matters**: This dramatically improves the API's performance and scalability. A faster response time leads to a better user experience and reduces the load on the database server.

### Bug 3: Non-Atomic Bulk Item Addition

*   **The Problem**: When adding multiple items in bulk, the code was committing each item to the database one by one inside a loop. If an error occurred midway, the database would be left in an inconsistent, partially updated state.
*   **The Fix**: I moved the database commit outside the loop, wrapping the entire operation in a single transaction. Now, either all items are added successfully, or none are.

    **Before (The Bug):**
    ```python
    # in app/services/item_service.py
    def bulk_add_items(db: Session, slot_id: str, entries: list[ItemBulkEntry]) -> int:
        # ...
        for e in entries:
            # ...
            db.add(item)
            db.commit() # <-- Commit is inside the loop, which is not atomic.
        return added
    ```

    **After (The Fix):**
    ```python
    # in app/services/item_service.py
    def bulk_add_items(db: Session, slot_id: str, entries: list[ItemBulkEntry]) -> int:
        # ...
        for e in entries:
            # ...
            db.add(item)
        
        db.commit() # <-- Commit is now outside the loop.
        return added_count
    ```
*   **Why This Matters**: This ensures data consistency. The database will never be left in a partially updated state from a failed bulk operation.

### Bug 4: Deleting a Non-Empty Slot

*   **The Problem**: The API allowed a slot to be deleted even if it still contained items. This could lead to "orphaned" items in the database that are no longer associated with any slot.
*   **The Fix**: I added a check to ensure a slot is empty before it can be deleted. If it's not, the API now returns a `400 Bad Request` error.

    **Before (The Bug):**
    ```python
    # in app/services/slot_service.py
    def delete_slot(db: Session, slot_id: str) -> None:
        slot = get_slot_by_id(db, slot_id)
        db.delete(slot) # <-- Deletes without checking if it has items.
        db.commit()
    ```

    **After (The Fix):**
    ```python
    # in app/services/slot_service.py
    def delete_slot(db: Session, slot_id: str) -> None:
        slot = get_slot_by_id(db, slot_id)
        if len(slot.items) > 0: # <-- New check here.
            raise ValueError("slot_not_empty")
        db.delete(slot)
        db.commit()
    ```
*   **Why This Matters**: This change protects data integrity and prevents accidental data loss.

### Bug 5: Incorrect Capacity Logic When Adding Items

*   **The Problem**: The logic for adding an item to a slot had a faulty check that compared the item count to an optional, global setting (`MAX_ITEMS_PER_SLOT`) instead of the slot's actual `capacity`.
*   **The Fix**: I removed the incorrect condition, so the validation is now correctly performed only against the specific `capacity` of the slot being modified.

    **Before (The Bug):**
    ```python
    # in app/services/item_service.py
    def add_item_to_slot(db: Session, slot_id: str, data: ItemCreate) -> Item:
        # ...
        if slot.current_item_count + data.quantity > slot.capacity:
            raise ValueError("capacity_exceeded")
        if slot.current_item_count + data.quantity < settings.MAX_ITEMS_PER_SLOT: # <-- Incorrect logic
            raise ValueError("capacity_exceeded")
    ```

    **After (The Fix):**
    ```python
    # in app/services/item_service.py
    def add_item_to_slot(db: Session, slot_id: str, data: ItemCreate) -> Item:
        # ...
        if slot.current_item_count + data.quantity > slot.capacity:
            raise ValueError("capacity_exceeded")
        # The incorrect logic has been removed.
    ```
*   **Why This Matters**: This ensures the application's logic is correct and predictable, preventing valid operations from failing unexpectedly.

### Bug 6: Price Update Timestamp Not Updating

*   **The Problem**: When an item's price was updated, the `updated_at` timestamp was being reset to its old value, meaning the change was not being tracked.
*   **The Fix**: I removed the line of code that was incorrectly resetting the timestamp. SQLAlchemy's `onupdate` configuration now correctly handles this automatically.

    **Before (The Bug):**
    ```python
    # in app/services/item_service.py
    def update_item_price(db: Session, item_id: str, price: int) -> None:
        # ...
        prev_updated = item.updated_at
        item.price = price
        item.updated_at = prev_updated # <-- Incorrectly resetting the timestamp.
        db.commit()
    ```

    **After (The Fix):**
    ```python
    # in app/services/item_service.py
    def update_item_price(db: Session, item_id: str, price: int) -> None:
        # ...
        item.price = price
        # The incorrect line was removed. SQLAlchemy now handles the timestamp.
        db.commit()
    ```
*   **Why This Matters**: Accurate timestamps are crucial for auditing, debugging, and tracking data changes over time.

---

## 2. Future Improvements for a Production-Ready Application

To make this project truly enterprise-grade, I would recommend the following next steps:

1.  **Implement Automated Testing**: The project lacks a test suite. I would add unit and integration tests to verify the logic, confirm these bug fixes are effective, and prevent future regressions.
2.  **Enhance Security with Authentication & Authorization**: The API is currently open. I would secure the administrative endpoints using a mechanism like **OAuth2 with JWT tokens**.
3.  **Refine the REST API Design**: To align more closely with best practices, I would refactor certain endpoints (e.g., `GET /slots/full-view` could become `GET /slots?view=full`).
