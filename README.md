from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)

# Database configuration
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///restaurant.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)


# =========================
# Database Models
# =========================

class MenuItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    price = db.Column(db.Float, nullable=False)
    category = db.Column(db.String(50), nullable=False)


class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    customer_name = db.Column(db.String(100), nullable=False)
    item_name = db.Column(db.String(100), nullable=False)
    quantity = db.Column(db.Integer, nullable=False)
    total_price = db.Column(db.Float, nullable=False)


# Create database tables
with app.app_context():
    db.create_all()


# =========================
# Home Route
# =========================

@app.route("/")
def home():
    return jsonify({
        "message": "Restaurant Management System",
        "status": "Application is running"
    })


# =========================
# Menu Routes
# =========================

@app.route("/menu", methods=["GET"])
def get_menu():
    items = MenuItem.query.all()

    menu = []

    for item in items:
        menu.append({
            "id": item.id,
            "name": item.name,
            "price": item.price,
            "category": item.category
        })

    return jsonify(menu)


@app.route("/menu", methods=["POST"])
def add_menu_item():
    data = request.get_json()

    if not data:
        return jsonify({"error": "JSON data is required"}), 400

    name = data.get("name")
    price = data.get("price")
    category = data.get("category")

    if not name or price is None or not category:
        return jsonify({
            "error": "name, price and category are required"
        }), 400

    try:
        price = float(price)
    except ValueError:
        return jsonify({
            "error": "price must be a number"
        }), 400

    item = MenuItem(
        name=name,
        price=price,
        category=category
    )

    db.session.add(item)
    db.session.commit()

    return jsonify({
        "message": "Menu item added successfully",
        "id": item.id
    }), 201


# =========================
# Order Routes
# =========================

@app.route("/orders", methods=["GET"])
def get_orders():
    orders = Order.query.all()

    result = []

    for order in orders:
        result.append({
            "id": order.id,
            "customer_name": order.customer_name,
            "item_name": order.item_name,
            "quantity": order.quantity,
            "total_price": order.total_price
        })

    return jsonify(result)


@app.route("/orders", methods=["POST"])
def create_order():
    data = request.get_json()

    if not data:
        return jsonify({"error": "JSON data is required"}), 400

    customer_name = data.get("customer_name")
    item_name = data.get("item_name")
    quantity = data.get("quantity")

    if not customer_name or not item_name or quantity is None:
        return jsonify({
            "error": "customer_name, item_name and quantity are required"
        }), 400

    try:
        quantity = int(quantity)
    except ValueError:
        return jsonify({
            "error": "quantity must be an integer"
        }), 400

    if quantity <= 0:
        return jsonify({
            "error": "quantity must be greater than 0"
        }), 400

    item = MenuItem.query.filter_by(name=item_name).first()

    if not item:
        return jsonify({
            "error": "Menu item not found"
        }), 404

    total_price = item.price * quantity

    order = Order(
        customer_name=customer_name,
        item_name=item.name,
        quantity=quantity,
        total_price=total_price
    )

    db.session.add(order)
    db.session.commit()

    return jsonify({
        "message": "Order created successfully",
        "order_id": order.id,
        "customer_name": customer_name,
        "item_name": item.name,
        "quantity": quantity,
        "total_price": total_price
    }), 201


# =========================
# Run Application
# =========================

if __name__ == "__main__":
    app.run(debug=True)
