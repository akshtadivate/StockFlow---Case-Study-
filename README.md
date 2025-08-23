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

````markdown

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



