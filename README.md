
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
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
        #output {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 10px;
            background-color: #f9f9f9;
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
            <h2>Menu</h2>
            <button onclick="showAdd()">Add Item</button>
            <button onclick="showRemove()">Remove Item</button>
            <button onclick="undoLastAction()">Undo Last Action</button>
            <button onclick="showPlaceOrder()">Place Order</button>
            <button onclick="processNextOrder()">Process Order</button>
            <button onclick="viewInventory()">View Inventory</button>
            <button onclick="viewOrders()">View Orders</button>
            <button onclick="exit()">Exit</button>
            <div id="output"></div>
            <div id="add" style="display: none;">
                <input type="text" id="itemName" placeholder="Item name">
                <input type="number" id="itemQty" placeholder="Quantity">
                <button onclick="addItem()">Add</button>
            </div>
            <div id="remove" style="display: none;">
                <input type="text" id="removeItemName" placeholder="Item name">
                <input type="number" id="removeItemQty" placeholder="Quantity">
                <button onclick="removeItem()">Remove</button>
            </div>
            <div id="placeOrder" style="display: none;">
                <input type="text" id="customerName" placeholder="Customer name">
                <input type="text" id="orderItemName" placeholder="Item name">
                <input type="number" id="orderQty" placeholder="Quantity">
                <button onclick="placeOrder()">Place Order</button>
            </div>
        </div>
    </div>

    <script>
        let inventory = [];
        let undoStack = [];
        let orderQueue = [];

        document.addEventListener("DOMContentLoaded", () => {
            output("Welcome to the Inventory Management System!");
        });

        function showAdd() {
            document.getElementById("add").style.display = "block";
        }

        function showRemove() {
            document.getElementById("remove").style.display = "block";
        }

        function showPlaceOrder() {
            document.getElementById("placeOrder").style.display = "block";
        }

        function addItem() {
            const name = document.getElementById("itemName").value;
            const qty = +document.getElementById("itemQty").value;
            let idx = inventory.findIndex(i => i.name === name);
            if (idx !== -1) {
                inventory[idx].quantity += qty;
            } else {
                inventory.push({ name, quantity: qty });
            }
            undoStack.push({ action: "add", item: name, qty });
            output(`Added ${qty} × ${name} to inventory.`);
        }

        function removeItem() {
            const name = document.getElementById("removeItemName").value;
            const qty = +document.getElementById("removeItemQty").value;
            let idx = inventory.findIndex(i => i.name === name);
            if (idx !== -1 && inventory[idx].quantity >= qty) {
                inventory[idx].quantity -= qty;
                if (inventory[idx].quantity === 0) {
                    inventory.splice(idx, 1);
                }
                undoStack.push({ action: "remove", item: name, qty });
                output(`Removed ${qty} × ${name} from inventory.`);
            } else {
                output("Insufficient stock or item not found!");
            }
        }

        function undoLastAction() {
            let act = undoStack.pop();
            if (!act) {
                output("No actions to undo.");
                return;
            }
            if (act.action === "add") {
                removeItemDirect(act.item, act.qty);
                output(`Undid ADD: Removed ${act.qty} × ${act.item}`);
            } else {
                addItemDirect(act.item, act.qty);
                output(`Undid REMOVE: Added back ${act.qty} × ${act.item}`);
            }
        }

        function placeOrder() {
            const customer = document.getElementById("customerName").value;
            const item = document.getElementById("orderItemName").value;
            const qty = +document.getElementById("orderQty").value;
            orderQueue.push({ customer, item, qty });
            output(`Order placed for ${customer}: ${qty} × ${item}`);
        }

        function processNextOrder() {
            let ord = orderQueue.shift();
            if (!ord) {
                output("No pending orders.");
                return;
            }
            output(`Processing order for ${ord.customer}: ${ord.qty} × ${ord.item}`);
            let idx = inventory.findIndex(i => i.name === ord.item);
            if (idx !== -1 && inventory[idx].quantity >= ord.qty) {
                removeItemDirect(ord.item, ord.qty);
                output("Order fulfilled!");
            } else {
                output("Order cannot be fulfilled - insufficient stock.");
            }
        }

        function viewInventory() {
            output(inventory.map(i => `• ${i.name}: ${i.quantity} units`).join("<br>"));
        }

        function viewOrders() {
            output(orderQueue.map(o => `• ${o.customer} → ${o.qty} × ${o.item}`).join("<br>"));
        }

        function output(msg) {
            document.getElementById("output").innerHTML = msg;
        }

        function exit() {
            window.close();
        }

        // Helper functions
        function removeItemDirect(name, qty) {
            let idx = inventory.findIndex(i => i.name === name);
            if (idx !== -1) {
                inventory[idx].quantity -= qty;
                if (inventory[idx].quantity === 0) {
                    inventory.splice(idx, 1);
                }
            }
        }

        function addItemDirect(name, qty) {
            let idx = inventory.findIndex(i => i.name === name);
            if (idx !== -1) {
                inventory[idx].quantity += qty;
            } else {
                inventory.push({ name, quantity: qty });
            }
        }
    </script>
</body>
</html>
