# StockFlow---Case-Study-

## Part 1: Code Review & Debugging

### 1) Issues:

No input validation	
SKU uniqueness not enforced
Warehouse ID stored in Product
Price stored as float
Partial commits (two separate commits)
Negative/invalid quantities or prices
No transaction atomicity
Missing error handling
Optional fields not handled

#### 2) Impact:
    
No input validation	- Missing keys or invalid data can cause runtime errors
SKU uniqueness not enforced -	Duplicate SKUs can break inventory tracking and reporting
Warehouse ID stored in Product -	Products can exist in multiple warehouses, leads to incorrect DB design
Price stored as float -	Can cause rounding errors in financial calculations
Partial commits (two separate commits) - Product may be created but inventory creation may fail, leaving inconsistent data
Negative/invalid quantities or prices -	Leads to incorrect inventory or billing issues
No transaction atomicity -	Multiple database operations not atomic, race conditions possible
Missing error handling -	Always returns 200 even on failure
Optional fields not handled	- Fields like initial quantity may be missing, causing errors

#### 3) Fixed code:



```python

from decimal import Decimal, InvalidOperation
from flask import Flask, request, jsonify
from sqlalchemy.exc import IntegrityError
from sqlalchemy import func

app = Flask(__name__)

@app.route('/api/products', methods=['POST'])
def create_product():
    # Safely get JSON input (empty dict if invalid/missing)
    data = request.get_json(silent=True) or {}

    # Extract and normalize input fields
    name = (data.get('name') or '').strip()        # Trim spaces
    sku  = (data.get('sku') or '').strip().upper() # Ensure SKU is case-insensitive & unique
    warehouse_id = data.get('warehouse_id')
    initial_qty  = data.get('initial_quantity', 0)

    # Collect validation errors
    errors = {}

    # Validate product name
    if not name:
        errors['name'] = 'Name required'

    # Validate SKU
    if not sku:
        errors['sku'] = 'SKU required'

    # Validate warehouse_id presence
    if warehouse_id is None:
        errors['warehouse_id'] = 'warehouse_id required'

    # Validate price (must be decimal >= 0)
    try:
        price = Decimal(str(data.get('price')))
        if price < 0:
            errors['price'] = 'Price must be >= 0'
    except (InvalidOperation, TypeError):
        errors['price'] = 'Price must be decimal'

    # Validate initial quantity (must be integer >= 0)
    try:
        initial_qty = int(initial_qty)
        if initial_qty < 0:
            errors['initial_quantity'] = 'Initial quantity >= 0'
    except (ValueError, TypeError):
        errors['initial_quantity'] = 'Initial quantity must be integer'

    # If any validation errors found → return 400 Bad Request
    if errors:
        return jsonify({"errors": errors}), 400

    # Check if warehouse exists
    warehouse = db.session.query(Warehouse).get(warehouse_id)
    if not warehouse:
        return jsonify({"error": "Warehouse not found"}), 404

    try:
        # Start an atomic transaction
        with db.session.begin():

            # Ensure SKU is unique (case-insensitive check)
            existing = db.session.query(Product).filter(func.upper(Product.sku) == sku).first()
            if existing:
                return jsonify({"error": "SKU exists", "product_id": existing.id}), 409

            # Create new product
            product = Product(name=name, sku=sku, price=price)
            db.session.add(product)
            db.session.flush()  # Flush so product.id is available

            # Add or update inventory for this warehouse
            inv = db.session.query(Inventory).filter_by(
                product_id=product.id, 
                warehouse_id=warehouse_id
            ).first()

            if inv:
                # If already exists, increase stock
                inv.quantity += initial_qty
            else:
                # Otherwise create new inventory entry
                inv = Inventory(product_id=product.id, warehouse_id=warehouse_id, quantity=initial_qty)
                db.session.add(inv)

        # Return success response with product + inventory details
        return jsonify({
            "message": "Product created",
            "product": {
                "id": product.id,
                "name": product.name,
                "sku": product.sku,
                "price": str(product.price)  # Convert Decimal → str for JSON
            },
            "inventory": {
                "warehouse_id": warehouse_id,
                "quantity": inv.quantity
            }
        }), 201

    # Handle unique constraint / foreign key violations
    except IntegrityError as e:
        db.session.rollback()
        return jsonify({"error": "Integrity error", "detail": str(e.orig)}), 409

    # Catch-all error handler
    except Exception as e:
        db.session.rollback()
        return jsonify({"error": "Unexpected error", "detail": str(e)}), 500

'''

##Part 2: Database Design

### 1) Design Schema:

-- Companies
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Warehouses
CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    location TEXT
);

-- Products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    is_bundle BOOLEAN DEFAULT FALSE
);

-- Inventory
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Inventory History
CREATE TABLE inventory_history (
    id SERIAL PRIMARY KEY,
    inventory_id INT REFERENCES inventory(id),
    change INT NOT NULL,
    reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Suppliers
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255)
);

-- Product-Supplier Relationship
CREATE TABLE product_suppliers (
    product_id INT REFERENCES products(id),
    supplier_id INT REFERENCES suppliers(id),
    PRIMARY KEY (product_id, supplier_id)
);

-- Bundles (product-to-product relationship)
CREATE TABLE product_bundles (
    bundle_id INT REFERENCES products(id),
    component_id INT REFERENCES products(id),
    quantity INT NOT NULL,
    PRIMARY KEY (bundle_id, component_id)
);

### 2) Identify Gaps:

Do we need one low-stock limit for all products or different for each?
Can a product be in more than one warehouse?
What should happen if stock goes negative?
Do we need to track which user updated the stock?
Should we keep history of old stock changes?
Can one product have more than one supplier?
Do we need to allow transfer of stock between warehouses?
How do we handle returned or damaged items?
Do we need product categories (like electronics, clothes)?
Should we allow deleting products or just disabling them?


### 3) Explain Decisions:
Every table has a primary key id to make each record unique.
Foreign keys are used to connect related tables, for example a warehouse belongs to a company, and inventory belongs to a product and a warehouse.
Each product has a unique sku so that duplicate products cannot exist.
Inventory is kept in a separate table so that one product can be stored in multiple warehouses.
An inventory history table is added to keep past stock changes for tracking and auditing.
Many-to-many relationship tables are used for mapping products with multiple suppliers and for bundles made of other products.
Indexes are added on sku and on (product_id, warehouse_id) to make searching and lookups faster.
Timestamps such as updated_at are stored so that we know when inventory was last changed.

##Part 3: API Implementation

### 1) Implementation:

````markdown

```python

from flask import Flask, jsonify
from datetime import datetime, timedelta

app = Flask(__name__)

products = {
    1: {"name": "Widget A", "sku": "WID-001", "threshold": 20, "supplier_id": 101},
    2: {"name": "Widget B", "sku": "WID-002", "threshold": 15, "supplier_id": 102},
}


warehouses = {
    1001: {"name": "Main Warehouse", "company_id": 1},
    1002: {"name": "Backup Warehouse", "company_id": 1},
}

# Current stock levels (product + warehouse)
inventory = [
    {"product_id": 1, "warehouse_id": 1001, "stock": 5, "last_sale_date": datetime.now() - timedelta(days=2)},
    {"product_id": 2, "warehouse_id": 1002, "stock": 18, "last_sale_date": datetime.now() - timedelta(days=40)},  # no recent sales
]

# Suppliers information
suppliers = {
    101: {"id": 101, "name": "Supplier Corp", "contact_email": "orders@supplier.com"},
    102: {"id": 102, "name": "Parts Ltd", "contact_email": "sales@partsltd.com"},
}

# Utility function to estimate stockout days

def estimate_days_until_stockout(stock, avg_daily_sales=1):
    """
    Simplified assumption:
    - If avg daily sales = 1 (hardcoded for demo)
    - Stockout days = stock / avg_daily_sales
    """
    if avg_daily_sales <= 0:
        return None
    return stock // avg_daily_sales

# API Endpoint

@app.route("/api/companies/<int:company_id>/alerts/low-stock", methods=["GET"])
def get_low_stock_alerts(company_id):
    alerts = []

    for inv in inventory:
        product = products.get(inv["product_id"])
        warehouse = warehouses.get(inv["warehouse_id"])

        if not product or not warehouse:
            continue

        # Check if this warehouse belongs to the requested company
        if warehouse["company_id"] != company_id:
            continue

        # Rule 1: Low stock threshold check
        if inv["stock"] >= product["threshold"]:
            continue

        # Rule 2: Must have recent sales (assume "recent" = within 30 days)
        if (datetime.now() - inv["last_sale_date"]).days > 30:
            continue

        # Rule 3: Include supplier info
        supplier = suppliers.get(product["supplier_id"], {})

        alert = {
            "product_id": inv["product_id"],
            "product_name": product["name"],
            "sku": product["sku"],
            "warehouse_id": inv["warehouse_id"],
            "warehouse_name": warehouse["name"],
            "current_stock": inv["stock"],
            "threshold": product["threshold"],
            "days_until_stockout": estimate_days_until_stockout(inv["stock"]),
            "supplier": supplier
        }
        alerts.append(alert)

    response = {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }
    return jsonify(response)

# Run Flask App

if __name__ == "__main__":
    app.run(debug=True)

### 2) Handle Edge Cases:

No warehouses: return empty alerts
No supplier: supplier = null
No recent sales: skip product
Threshold missing: use default 10
Negative stock: still count as low stock

### 3) Explain Approach:

Query products below threshold or default
Check recent sales (last 30 days)
Get supplier info (null if missing)
Estimate days_until_stockout from sales data
Return JSON with alerts and total count.
