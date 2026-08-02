<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Inventory Management System</title>
<style>
    body {
        font-family: Arial, sans-serif;
        background-color: #f0f0f0;
    }
    .container {
        width: 80%;
        margin: 40px auto;
        padding: 20px;
        background-color: #fff;
        border: 1px solid #ddd;
        border-radius: 10px;
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    }
    #menu {
        text-align: center;
    }
    .panel {
        display: none;
        margin: 15px auto;
        padding: 15px;
        max-width: 400px;
        border: 1px solid #ddd;
        border-radius: 10px;
        background-color: #f9f9f9;
    }
    .panel input {
        display: block;
        width: 100%;
        box-sizing: border-box;
        margin: 6px 0;
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 5px;
    }
    #output {
        margin-top: 20px;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 10px;
        background-color: #f9f9f9;
        min-height: 24px;
        white-space: pre-line;
    }
    #output.error {
        color: #b00020;
        background-color: #fdecea;
        border-color: #f5c2c0;
    }
    #output.success {
        color: #1e4620;
        background-color: #eaf7ea;
        border-color: #c3e6c3;
    }
    button {
        margin: 10px;
        padding: 10px 20px;
        border: none;
        border-radius: 5px;
        background-color: #4CAF50;
        color: #fff;
        cursor: pointer;
    }
    button:hover {
        background-color: #3e8e41;
    }
</style>
</head>
<body>
<div class="container">
    <h1>Inventory Management System</h1>
    <div id="menu">
        <h3>Menu</h3>
        <button onclick="showAdd()">Add Item</button>
        <button onclick="showRemove()">Remove Item</button>
        <button onclick="undoLastAction()">Undo Last Action</button>
        <button onclick="showPlaceOrder()">Place Order</button>
        <button onclick="processNextOrder()">Process Order</button>
        <button onclick="viewInventory()">View Inventory</button>
        <button onclick="viewOrders()">View Orders</button>
        <button onclick="exitApp()">Exit</button>
    </div>

    <div id="add" class="panel">
        <h3>Add Item</h3>
        <input type="text" id="itemName" placeholder="Item name">
        <input type="number" id="itemQty" placeholder="Quantity" min="1" step="1">
        <button onclick="addItem()">Confirm Add</button>
    </div>

    <div id="remove" class="panel">
        <h3>Remove Item</h3>
        <input type="text" id="removeItemName" placeholder="Item name">
        <input type="number" id="removeItemQty" placeholder="Quantity" min="1" step="1">
        <button onclick="removeItem()">Confirm Remove</button>
    </div>

    <div id="placeOrder" class="panel">
        <h3>Place Order</h3>
        <input type="text" id="customerName" placeholder="Customer name">
        <input type="text" id="orderItemName" placeholder="Item name">
        <input type="number" id="orderQty" placeholder="Quantity" min="1" step="1">
        <button onclick="placeOrder()">Confirm Order</button>
    </div>

    <div id="output"></div>
</div>

<script>
    const STORAGE_KEY = "inventory_app_state_v1";

    let inventory = [];
    let undoStack = [];
    let orderQueue = [];

    document.addEventListener("DOMContentLoaded", () => {
        loadState();
        output("Welcome to the Inventory Management System!", "success");
    });

    // ---------- Persistence ----------
    function saveState() {
        try {
            const state = { inventory, undoStack, orderQueue };
            localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
        } catch (e) {
            // localStorage may be unavailable (e.g. private browsing quota) -
            // fail silently so the app still works in-memory for the session.
            console.warn("Could not save state:", e);
        }
    }

    function loadState() {
        try {
            const raw = localStorage.getItem(STORAGE_KEY);
            if (!raw) return;
            const state = JSON.parse(raw);
            if (Array.isArray(state.inventory)) inventory = state.inventory;
            if (Array.isArray(state.undoStack)) undoStack = state.undoStack;
            if (Array.isArray(state.orderQueue)) orderQueue = state.orderQueue;
        } catch (e) {
            console.warn("Could not load saved state:", e);
        }
    }

    // ---------- Panel toggles ----------
    function hideAllPanels() {
        document.querySelectorAll(".panel").forEach(p => p.style.display = "none");
    }

    function showAdd() {
        hideAllPanels();
        document.getElementById("add").style.display = "block";
    }

    function showRemove() {
        hideAllPanels();
        document.getElementById("remove").style.display = "block";
    }

    function showPlaceOrder() {
        hideAllPanels();
        document.getElementById("placeOrder").style.display = "block";
    }

    // ---------- Validation helpers ----------
    function normalizeName(raw) {
        return raw.trim().replace(/\s+/g, " ");
    }

    // Case-insensitive lookup so "Box" and "box" are treated as the same item.
    function findItemIndex(name) {
        const key = name.toLowerCase();
        return inventory.findIndex(i => i.name.toLowerCase() === key);
    }

    function parseQty(raw) {
        if (raw === "" || raw === null) return null;
        const n = Number(raw);
        if (!Number.isFinite(n) || !Number.isInteger(n) || n <= 0) return null;
        return n;
    }

    // ---------- Core actions ----------
    function addItem() {
        const name = normalizeName(document.getElementById("itemName").value);
        const qty = parseQty(document.getElementById("itemQty").value);

        if (!name) {
            output("Please enter an item name.", "error");
            return;
        }
        if (qty === null) {
            output("Please enter a valid positive whole number for quantity.", "error");
            return;
        }

        let idx = findItemIndex(name);
        if (idx !== -1) {
            inventory[idx].quantity += qty;
        } else {
            inventory.push({ name, quantity: qty });
        }
        undoStack.push({ action: "add", item: name, qty });
        saveState();
        output(`Added ${qty} × ${name} to inventory.`, "success");
        clearInputs("itemName", "itemQty");
    }

    function removeItem() {
        const name = normalizeName(document.getElementById("removeItemName").value);
        const qty = parseQty(document.getElementById("removeItemQty").value);

        if (!name) {
            output("Please enter an item name.", "error");
            return;
        }
        if (qty === null) {
            output("Please enter a valid positive whole number for quantity.", "error");
            return;
        }

        let idx = findItemIndex(name);
        if (idx === -1) {
            output(`"${name}" was not found in inventory.`, "error");
            return;
        }
        if (inventory[idx].quantity < qty) {
            output(`Insufficient stock: only ${inventory[idx].quantity} × ${inventory[idx].name} available.`, "error");
            return;
        }

        inventory[idx].quantity -= qty;
        const actualName = inventory[idx].name;
        if (inventory[idx].quantity === 0) {
            inventory.splice(idx, 1);
        }
        undoStack.push({ action: "remove", item: actualName, qty });
        saveState();
        output(`Removed ${qty} × ${actualName} from inventory.`, "success");
        clearInputs("removeItemName", "removeItemQty");
    }

    function undoLastAction() {
        let act = undoStack.pop();
        if (!act) {
            output("No actions to undo.", "error");
            return;
        }
        if (act.action === "add") {
            removeItemDirect(act.item, act.qty);
            output(`Undid ADD: Removed ${act.qty} × ${act.item}`, "success");
        } else {
            addItemDirect(act.item, act.qty);
            output(`Undid REMOVE: Added back ${act.qty} × ${act.item}`, "success");
        }
        saveState();
    }

    function placeOrder() {
        const customer = normalizeName(document.getElementById("customerName").value);
        const item = normalizeName(document.getElementById("orderItemName").value);
        const qty = parseQty(document.getElementById("orderQty").value);

        if (!customer) {
            output("Please enter a customer name.", "error");
            return;
        }
        if (!item) {
            output("Please enter an item name.", "error");
            return;
        }
        if (qty === null) {
            output("Please enter a valid positive whole number for quantity.", "error");
            return;
        }

        orderQueue.push({ customer, item, qty });
        saveState();
        output(`Order placed for ${customer}: ${qty} × ${item}`, "success");
        clearInputs("customerName", "orderItemName", "orderQty");
    }

    function processNextOrder() {
        let ord = orderQueue.shift();
        if (!ord) {
            output("No pending orders.", "error");
            return;
        }
        let idx = findItemIndex(ord.item);
        if (idx !== -1 && inventory[idx].quantity >= ord.qty) {
            removeItemDirect(inventory[idx].name, ord.qty);
            saveState();
            output(`Processing order for ${ord.customer}: ${ord.qty} × ${ord.item}\nOrder fulfilled!`, "success");
        } else {
            // Put the order back at the front since it couldn't be fulfilled.
            orderQueue.unshift(ord);
            saveState();
            output(`Processing order for ${ord.customer}: ${ord.qty} × ${ord.item}\nOrder cannot be fulfilled - insufficient stock.`, "error");
        }
    }

    function viewInventory() {
        if (inventory.length === 0) {
            output("Inventory is empty.", "success");
            return;
        }
        const lines = inventory.map(i => `• ${i.name}: ${i.quantity} units`).join("\n");
        output(lines, "success");
    }

    function viewOrders() {
        if (orderQueue.length === 0) {
            output("No pending orders.", "success");
            return;
        }
        const lines = orderQueue.map(o => `• ${o.customer} → ${o.qty} × ${o.item}`).join("\n");
        output(lines, "success");
    }

    // Safe output: uses textContent so any item/customer name typed by a user
    // can never be interpreted as HTML/script (prevents XSS).
    function output(msg, kind) {
        const el = document.getElementById("output");
        el.textContent = msg;
        el.classList.remove("error", "success");
        if (kind) el.classList.add(kind);
    }

    function exitApp() {
        hideAllPanels();
        output("Session ended. You can close this tab; your data has been saved.", "success");
    }

    function clearInputs(...ids) {
        ids.forEach(id => {
            const el = document.getElementById(id);
            if (el) el.value = "";
        });
    }

    // ---------- Helper functions (used by undo/order processing) ----------
    function removeItemDirect(name, qty) {
        let idx = findItemIndex(name);
        if (idx !== -1) {
            inventory[idx].quantity -= qty;
            if (inventory[idx].quantity <= 0) {
                inventory.splice(idx, 1);
            }
        }
    }

    function addItemDirect(name, qty) {
        let idx = findItemIndex(name);
        if (idx !== -1) {
            inventory[idx].quantity += qty;
        } else {
            inventory.push({ name, quantity: qty });
        }
    }
</script>
</body>
</html>
