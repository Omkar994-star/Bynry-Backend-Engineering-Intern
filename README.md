# Bynry-Backend-Engineering-Intern
Part 1: Code Review & Debugging

Identified Issues:

	1:No Input Validation
	data = request.json
Assumes all required fields exist
No type checking (e.g., price could be string/null)
      Impact:
Crashes with KeyError
Invalid data stored (e.g., negative price, missing SKU)
	
	2:SKU Uniqueness Not Enforced
No check if sku already exists
      Impact:
Duplicate SKUs lead to inventory confusion and order issues
	
	3:No Error Handling
No try/except block
      Impact:
API may crash and return 500 errors
No meaningful error response to the client

	4:Price Not Handled as Decimal
Likely stored as float
      Impact:
Floating point precision errors (e.g., 19.999999)
Financial inaccuracies

	5:No Validation of Warehouse Existence
warehouse_id may not exist
      Impact:
Can cause foreign key errors or invalid references

	6:No Response Status Code
	       return {"message": "Product created"}
       Impact:
Client cannot know if request failed or succeeded
Always returns HTTP 200 

	7:Optional Fields Not Handled
Assumes initial_quantity is always present
      Impact:
KeyError if the field is missing

	8:No Transaction Handling (Atomicity Issue)
		db.session.commit() # product
		...
		db.session.commit() # inventory
If the first commit succeeds but the second fails:
Product is saved
Inventory is NOT saved
This creates inconsistent data (partial/half-created records).
 Impact:
Product exists but has no inventory
Orders may fail or show incorrect stock


9:Hardcoded Warehouse Logic
	warehouse_id = data['warehouse_id']
Impact:
Assumes one warehouse per product
Does not support multiple warehouses per product

Corrected Version:
from flask import request, jsonify
from decimal import Decimal, InvalidOperation
from sqlalchemy.exc import IntegrityError
from app import db
from models import Product, Inventory, Warehouse

@app.route('/api/products', methods=['POST'])
def create_product():
    try:
        data = request.get_json()

        # 1. Validate required fields
        required_fields = ['name', 'sku', 'price', 'warehouse_id']
        for field in required_fields:
            if field not in data:
                return jsonify({"error": f"{field} is required"}), 400

        # 2. Validate price
        try:
            price = Decimal(str(data['price']))
            if price < 0:
                return jsonify({"error": "Price must be non-negative"}), 400
        except:
            return jsonify({"error": "Invalid price"}), 400

        # 3. Validate quantity (optional)
        quantity = data.get('initial_quantity', 0)
        if not isinstance(quantity, int) or quantity < 0:
            return jsonify({"error": "Invalid quantity"}), 400

        # 4. Check SKU uniqueness
        if Product.query.filter_by(sku=data['sku']).first():
            return jsonify({"error": "SKU already exists"}), 409

        # 5. Check warehouse exists
        warehouse = Warehouse.query.get(data['warehouse_id'])
        if not warehouse:
            return jsonify({"error": "Warehouse not found"}), 404

        # 6. Single transaction (fix atomicity issue)
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price,
            warehouse_id=data['warehouse_id']
        )

        db.session.add(product)
        db.session.flush()  # get product.id without committing

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=quantity
        )

        db.session.add(inventory)

        db.session.commit()

        # 7. Proper response
        return jsonify({
            "message": "Product created",
            "product_id": product.id
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "Database error"}), 400
 except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500
1. Input Validation
Added:
data = request.get_json()
Now:
Checks required fields before using them
Prevents crashes like KeyError when fields are missing

2. SKU Uniqueness
Added:
Product.query.filter_by(sku=data['sku']).first()
Explanation:
Checks if a product with the same SKU already exists
Prevents duplicate SKUs in the database

3. Error Handling
Added:
try / except
Explanation:
Wraps the entire logic in error handling
Prevents API crashes
Returns proper error messages instead of server failure

4. Price Handling
Before:
price = data['price']
Now:
price = Decimal(str(data['price']))
Explanation:
Converts price to Decimal instead of float
Avoids floating point precision errors
Ensures accurate financial calculations

5. Warehouse Validation
Added:
Warehouse.query.get(data['warehouse_id'])
Explanation:
Checks if the warehouse exists in the database
Prevents invalid foreign key references

6. Status Code Fix
Before:
return {"message": "Product created"}
Now:
return jsonify(...), 201
Explanation:
Returns proper HTTP status code (201 = Created)
Helps client understand request success/failure

7. Optional Field Handling
Before:
data['initial_quantity']
Now:
data.get('initial_quantity', 0)
Explanation:
Uses default value if field is missing
Prevents KeyError
Makes field optional

8. Transaction Fix (Most Important)
Before:
commit()  # product
commit()  # inventory
Problem:
Product may be saved successfully
Inventory creation may fail
Leads to inconsistent database (partial data)
Now:
flush()
commit()  # only once
Explanation:
Uses a single transaction
Ensures both product and inventory are saved together
If any step fails → everything is rolled back

9:Hardcoded Warehouse Logic
Current Behavior:
One product is linked to only one warehouse
Limitation:
Does not support multiple warehouses per product
Future Improvement:
Allow multiple inventory entries:
"inventories": [
 {"warehouse_id": 1, "quantity": 10},
 {"warehouse_id": 2, "quantity": 5}
]
Part 2: Database Design

Design Schema:

Companies
CREATE TABLE companies (
 id BIGSERIAL PRIMARY KEY,
 name TEXT NOT NULL,
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
 );

Warehouses
CREATE TABLE warehouses (
 id BIGSERIAL PRIMARY KEY,
 company_id BIGINT NOT NULL,
 name TEXT NOT NULL,
 location TEXT,
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
FOREIGN KEY (company_id) REFERENCES companies(id) 
);

Products
CREATE TABLE products (
 id BIGSERIAL PRIMARY KEY,
 company_id BIGINT NOT NULL,
 name TEXT NOT NULL,
 sku TEXT NOT NULL,
 price DECIMAL(12,2) NOT NULL,
 is_bundle BOOLEAN DEFAULT FALSE,
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
UNIQUE (sku),  -- or (company_id, sku) depending on requirement 
FOREIGN KEY (company_id) REFERENCES companies(id) 
);

Inventory (Current State)
CREATE TABLE inventory (
 id BIGSERIAL PRIMARY KEY,
 product_id BIGINT NOT NULL,
 warehouse_id BIGINT NOT NULL,
 quantity INTEGER NOT NULL DEFAULT 0,
 updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
UNIQUE (product_id, warehouse_id), 
FOREIGN KEY (product_id) REFERENCES products(id), 
FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) 
);

Inventory History (Audit Trail)
CREATE TABLE inventory_transactions (
 id BIGSERIAL PRIMARY KEY,
 product_id BIGINT NOT NULL,
 warehouse_id BIGINT NOT NULL,
 change_quantity INTEGER NOT NULL,
 transaction_type TEXT, -- e.g. 'IN', 'OUT', 'ADJUSTMENT'
 reference_id TEXT, -- order_id, shipment_id, etc.
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
FOREIGN KEY (product_id) REFERENCES products(id), 
FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) 
);

Suppliers
CREATE TABLE suppliers (
 id BIGSERIAL PRIMARY KEY,
 name TEXT NOT NULL,
 contact_info TEXT,
 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
 );


Supplier → Products (Many-to-Many)
CREATE TABLE supplier_products (
 supplier_id BIGINT NOT NULL,
 product_id BIGINT NOT NULL,
 cost_price DECIMAL(12,2),
 lead_time_days INTEGER,
PRIMARY KEY (supplier_id, product_id), 
FOREIGN KEY (supplier_id) REFERENCES suppliers(id), 
FOREIGN KEY (product_id) REFERENCES products(id) 
);

Product Bundles (Composition Table)
CREATE TABLE product_bundles (
 bundle_id BIGINT NOT NULL,
 component_product_id BIGINT NOT NULL,
 quantity INTEGER NOT NULL,
PRIMARY KEY (bundle_id, component_product_id), 
FOREIGN KEY (bundle_id) REFERENCES products(id), 
FOREIGN KEY (component_product_id) REFERENCES products(id) 
);


MISSING REQUIREMENTS:
SKU Scope
Is SKU globally unique or unique per company?

Inventory Rules
Can inventory go negative?
Do we need reserved stock (for orders not yet shipped)?
Do we track damaged or returned stock separately?

Inventory Tracking
What triggers inventory changes?
Orders
Manual adjustments
Supplier shipments

Bundles
Can bundles contain other bundles (nested bundles)?
Should bundle price be derived or stored?

Suppliers
Can multiple suppliers supply the same product? (likely yes)
Do we track purchase orders?

Multi-Tenancy
Can users access multiple companies?
Should data be strictly isolated per company?

Pricing
Is price fixed or does it vary by warehouse or region?
Do we need discounting or dynamic pricing?

Auditing
Do we need to track who made the change (user_id)?

Performance
Expected scale:
Millions of products?
High-frequency inventory updates?

DESIGN DECISIONS:

Separate Inventory vs Transactions
Why?
inventory = current state (fast reads)
inventory_transactions = history (audit + analytics)
Avoids:
Expensive aggregation queries for every read

Composite Unique Key
UNIQUE (product_id, warehouse_id)
Ensures:
One row per product per warehouse
Prevents duplicate inventory records

Many-to-Many Relationships
supplier_products
product_bundles
Benefits:
Flexible and scalable design

Bundle Design Choice
Used self-referencing table
Supports:
Complex product compositions
Future extensibility (nested bundles if allowed)

Indexing Strategy
Add indexes for performance:
CREATE INDEX idx_inventory_product ON inventory(product_id);
 CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_transactions_product ON inventory_transactions(product_id);
 CREATE INDEX idx_transactions_warehouse ON inventory_transactions(warehouse_id);
 CREATE INDEX idx_transactions_created_at ON inventory_transactions(created_at);
Improves:
Inventory lookup
Reporting queries
Time-based analysis

Foreign Keys Everywhere
Ensures:
Referential integrity
No orphan records

Decimal for Money
DECIMAL(12,2)
Avoids:
Floating-point precision errors



Part 3: API Implementation

Since requirements are incomplete, I’m assuming:
products has a product_type field
product_types table stores low_stock_threshold
“Recent sales” = sales in last 30 days
Sales stored in order_items + orders
supplier_products links products to suppliers
One primary supplier per product (simplified)
days_until_stockout = current_stock / avg_daily_sales

IMPLEMENTATION:
   
	from flask import jsonify
	from datetime import datetime, timedelta
	from sqlalchemy import func

	@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
	def get_low_stock_alerts(company_id):
	    """
	    Returns low-stock alerts for a company.
    Logic:
    1. Get products with recent sales (last 30 days)
    2. Calculate avg daily sales per product per warehouse
    3. Compare inventory with threshold
    4. Attach supplier info
    """

    try:
        # 🔹 Configurable window
        DAYS_WINDOW = 30
        recent_date = datetime.utcnow() - timedelta(days=DAYS_WINDOW)

        # 🔹 Step 1: Get recent sales per product per warehouse
        sales_subquery = (
            db.session.query(
                OrderItem.product_id,
                Order.warehouse_id,
                func.sum(OrderItem.quantity).label("total_sold")
            )
            .join(Order, Order.id == OrderItem.order_id)
            .filter(Order.created_at >= recent_date)
            .group_by(OrderItem.product_id, Order.warehouse_id)
            .subquery()
        )

        # 🔹 Step 2: Main query
        results = (
            db.session.query(
                Product.id.label("product_id"),
                Product.name.label("product_name"),
                Product.sku,
                Warehouse.id.label("warehouse_id"),
                Warehouse.name.label("warehouse_name"),
                Inventory.quantity.label("current_stock"),
                ProductType.low_stock_threshold.label("threshold"),
                sales_subquery.c.total_sold,
                Supplier.id.label("supplier_id"),
                Supplier.name.label("supplier_name"),
                Supplier.contact_email
            )
            .join(Inventory, Inventory.product_id == Product.id)
            .join(Warehouse, Warehouse.id == Inventory.warehouse_id)
            .join(ProductType, ProductType.id == Product.product_type)
            .join(sales_subquery,
                  (sales_subquery.c.product_id == Product.id) &
                  (sales_subquery.c.warehouse_id == Warehouse.id))
            .outerjoin(SupplierProduct, SupplierProduct.product_id == Product.id)
            .outerjoin(Supplier, Supplier.id == SupplierProduct.supplier_id)
            .filter(Product.company_id == company_id)
            .all()
        )

        alerts = []

        for row in results:
            # 🔹 Step 3: Calculate avg daily sales
            avg_daily_sales = (row.total_sold or 0) / DAYS_WINDOW

            # Skip if no meaningful sales
            if avg_daily_sales <= 0:
                continue

            # 🔹 Step 4: Check low stock condition
            if row.current_stock >= row.threshold:
                continue

            # 🔹 Step 5: Calculate days until stockout
            days_until_stockout = int(row.current_stock / avg_daily_sales) \
                if avg_daily_sales > 0 else None

            alerts.append({
                "product_id": row.product_id,
                "product_name": row.product_name,
                "sku": row.sku,
                "warehouse_id": row.warehouse_id,
                "warehouse_name": row.warehouse_name,
                "current_stock": row.current_stock,
                "threshold": row.threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": row.supplier_id,
                    "name": row.supplier_name,
                    "contact_email": row.contact_email
                } if row.supplier_id else None
            })

        return jsonify({
            "alerts": alerts,
            "total_alerts": len(alerts)
        }), 200

    except Exception as e:
        return jsonify({"error": str(e)}), 500



EDGE CASES HANDLED:

No Sales Data
if avg_daily_sales <= 0:
 continue
Avoids divide-by-zero errors
Skips irrelevant alerts for products with no recent sales

Missing Supplier
if row.supplier_id else None
Prevents crashes when supplier data is not available
Ensures safe handling of optional relationships

Zero or Negative Stock
Still triggers low stock alert
days_until_stockout becomes 0 or None
Ensures critical cases are not ignored

Large Data Volume
Uses aggregation in database (SUM, GROUP BY) instead of Python
Improves performance and scalability
Reduces memory usage and processing time



APPROACH EXPLAINED:

Filter Only “Active Products”
Uses sales_subquery
Ensures only products with recent sales are considered
Benefits:
Avoids noise from inactive or dead inventory
Focuses only on relevant products

Warehouse-Level Granularity
Grouped by:
product_id
warehouse_id
Reason:
Same product can have different stock levels in different warehouses
Enables accurate, location-specific alerts

Threshold via Product Type
Uses ProductType.low_stock_threshold
Benefits:
Flexible threshold management
Supports different rules for different categories (e.g., electronics vs FMCG)

Days Until Stockout
Formula:
 current_stock / avg_daily_sales
Benefits:
Provides actionable insight
Helps predict when stock will run out instead of just flagging low stock

Supplier Join
Uses outerjoin(SupplierProduct)
Benefits:
Ensures alerts are still generated even if supplier information is missing
Maintains completeness of alert system
