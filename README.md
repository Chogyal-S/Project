# Project
CREATE DATABASE IF NOT EXISTS agri_market;
USE agri_market;

DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS cart;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS users;


CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(120) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('buyer','admin') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(120) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    unit VARCHAR(30) DEFAULT 'kg',
    quantity VARCHAR(50) NOT NULL,
    location VARCHAR(120) NOT NULL,
    description TEXT NOT NULL,
    image VARCHAR(255) NOT NULL,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL
);

CREATE TABLE cart (
    id INT AUTO_INCREMENT PRIMARY KEY,
    buyer_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT DEFAULT 1,
    FOREIGN KEY (buyer_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    buyer_id INT NOT NULL,
    buyer_name VARCHAR(100),
    address TEXT,
    payment_method VARCHAR(50),
    status VARCHAR(50) DEFAULT 'New Order',
    viewed TINYINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (buyer_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT,
    product_name VARCHAR(120),
    price DECIMAL(10,2),
    quantity INT,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

 <?php
session_start();

$host = "localhost";
$user = "root";
$pass = "";
$dbname = "agri_market";

$conn = new mysqli($host, $user, $pass, $dbname);

if ($conn->connect_error) {
    die("Database connection failed: " . $conn->connect_error);
}

function is_logged_in() {
    return isset($_SESSION['user_id']);
}

function is_buyer() {
    return isset($_SESSION['role']) && $_SESSION['role'] === 'buyer';
}

function is_admin() {
    return isset($_SESSION['role']) && $_SESSION['role'] === 'admin';
}

function require_login() {
    if (!is_logged_in()) {
        header("Location: login.php");
        exit();
    }
}

function require_admin() {
    require_login();
    if (!is_admin()) {
        header("Location: index.php");
        exit();
    }
}

function require_buyer() {
    require_login();
    if (!is_buyer()) {
        header("Location: index.php");
        exit();
    }
}

function order_count($conn) {
    if (!is_admin()) return 0;
    $result = $conn->query("SELECT COUNT(*) AS total FROM orders WHERE viewed = 0");
    $row = $result->fetch_assoc();
    return $row['total'];
}

function cart_count($conn) {
    if (!is_buyer()) return 0;
    $buyer_id = $_SESSION['user_id'];
    $stmt = $conn->prepare("SELECT COALESCE(SUM(quantity),0) AS total FROM cart WHERE buyer_id=?");
    $stmt->bind_param("i", $buyer_id);
    $stmt->execute();
    $row = $stmt->get_result()->fetch_assoc();
    return $row['total'];
}
?>
<?php require_once "config.php"; ?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Agri Market</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="style.css">
</head>
<body>
<header class="navbar">
    <a class="logo" href="index.php">Agri <span>Market</span></a>

    <button class="menu-toggle" onclick="document.querySelector('.nav').classList.toggle('nav-open')">☰</button>

    <nav class="nav">
        <a href="index.php">Home</a>
        <a href="marketplace.php">Marketplace</a>

        <?php if (is_buyer()): ?>
            <a href="buyer_dashboard.php">Buyer Dashboard</a>
            <a href="cart.php">🛒 Cart <span class="badge"><?php echo cart_count($conn); ?></span></a>
            <a href="buyer_orders.php">My Orders</a>
        <?php endif; ?>

        <?php if (is_admin()): ?>
            <a href="admin_dashboard.php">Dashboard</a>
            <a href="add_product.php">Add Produce</a>
            <a href="orders.php">Orders <span class="badge"><?php echo order_count($conn); ?></span></a>
            <a href="reports.php">Reports</a>
        <?php endif; ?>

        <?php if (!is_logged_in()): ?>
            <a href="login.php">Login</a>
            <a href="signup.php">Create Account</a>
        <?php endif; ?>
    </nav>
</header>

<?php if (is_logged_in()): ?>
    <button class="floating-logout-btn" onclick="confirmLogout()">⎋ Logout</button>
    <script>
        function confirmLogout() {
            if (confirm("Are you sure you want to log out?")) {
                window.location.href = "logout.php";
            }
        }
    </script>
<?php endif; ?>
<footer class="footer">
    <p><b>Agri Market</b></p>
    <p>
        <a href="about.php">About</a>
        <a href="support.php">Support</a>
        <a href="how_it_works.php">How It Works</a>
        <a href="contact.php">Contact</a>
    </p>
</footer>
</body>
</html>
<?php include "header.php"; ?>
<section class="hero">
    <div>
        <h1>Fresh Farm Produce, Connected Fresh</h1>
        <p>Agri Market helps farmers and buyers connect through a real PHP and MySQL web platform for listing produce, managing orders, and improving market access.</p>
        <a class="btn" href="marketplace.php">Explore Marketplace</a>
    </div>
    <div class="hero-card">
        <img src="assets/veg-mix.png">
        <h2>Fresh. Fair. Connected.</h2>
        <p>A modern platform for farmers, buyers, agents and administrators.</p>
    </div>
</section>
<?php include "footer.php"; ?>
<?php
include "config.php";
$message = "";

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $name = trim($_POST["full_name"]);
    $email = trim($_POST["email"]);
    $password = password_hash($_POST["password"], PASSWORD_DEFAULT);
    $role = $_POST["role"];

    $stmt = $conn->prepare("INSERT INTO users (full_name, email, password, role) VALUES (?, ?, ?, ?)");
    $stmt->bind_param("ssss", $name, $email, $password, $role);

    if ($stmt->execute()) {
        $message = "Account created successfully. You can now login.";
    } else {
        $message = "Error: Email may already be registered.";
    }
}
include "header.php";
?>
<section class="page-header"><h1>Create Account</h1><p>Create a buyer or farmer/admin account.</p></section>
<section class="section">
<form method="POST">
    <p style="color:green;font-weight:bold;"><?php echo $message; ?></p>
    <label>Full Name / Farm Name</label>
    <input name="full_name" required>
    <label>Email</label>
    <input type="email" name="email" required>
    <label>Password</label>
    <input type="password" name="password" required>
    <label>Account Type</label>
    <select name="role" required>
        <option value="buyer">Buyer</option>
        <option value="admin">Farmer/Admin</option>
    </select>
    <button class="btn" type="submit">Create Account</button>
    <a class="btn btn-light" href="login.php">Already have an account?</a>
</form>
</section>
<?php include "footer.php"; ?>
<?php
include "config.php";
$message = "";

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $name = trim($_POST["full_name"]);
    $email = trim($_POST["email"]);
    $password = password_hash($_POST["password"], PASSWORD_DEFAULT);
    $role = $_POST["role"];

    $stmt = $conn->prepare("INSERT INTO users (full_name, email, password, role) VALUES (?, ?, ?, ?)");
    $stmt->bind_param("ssss", $name, $email, $password, $role);

    if ($stmt->execute()) {
        $message = "Account created successfully. You can now login.";
    } else {
        $message = "Error: Email may already be registered.";
    }
}
include "header.php";
?>
<section class="page-header"><h1>Create Account</h1><p>Create a buyer or farmer/admin account.</p></section>
<section class="section">
<form method="POST">
    <p style="color:green;font-weight:bold;"><?php echo $message; ?></p>
    <label>Full Name / Farm Name</label>
    <input name="full_name" required>
    <label>Email</label>
    <input type="email" name="email" required>
    <label>Password</label>
    <input type="password" name="password" required>
    <label>Account Type</label>
    <select name="role" required>
        <option value="buyer">Buyer</option>
        <option value="admin">Farmer/Admin</option>
    </select>
    <button class="btn" type="submit">Create Account</button>
    <a class="btn btn-light" href="login.php">Already have an account?</a>
</form>
</section>
<?php include "footer.php"; ?>
<?php include "header.php"; ?>
<section class="page-header"><h1>Marketplace</h1><p>Browse fresh produce. Buyers can add to cart, while admins can edit and delete products.</p></section>
<section class="section">
    <form method="GET" class="search-box">
        <input type="text" name="search" placeholder="Search by product, location or description" value="<?php echo htmlspecialchars($_GET['search'] ?? ''); ?>">
        <button class="btn" type="submit">Search</button>
        <a class="btn btn-light" href="marketplace.php">Clear</a>
    </form>
</section>
<section class="section" style="padding-top:0;">
<div class="marketplace-grid">
<?php
$search = $_GET['search'] ?? '';
if ($search) {
    $like = "%$search%";
    $stmt = $conn->prepare("SELECT * FROM products WHERE name LIKE ? OR location LIKE ? OR description LIKE ? ORDER BY created_at DESC");
    $stmt->bind_param("sss", $like, $like, $like);
} else {
    $stmt = $conn->prepare("SELECT * FROM products ORDER BY created_at DESC");
}
$stmt->execute();
$result = $stmt->get_result();

while ($p = $result->fetch_assoc()):
?>
    <div class="card product-card">
        <div class="product-image"><img src="<?php echo htmlspecialchars($p['image']); ?>"></div>
        <span class="tag">Fresh Produce</span>
        <h2><?php echo htmlspecialchars($p['name']); ?></h2>
        <p><?php echo htmlspecialchars($p['description']); ?></p>
        <p><b>Location:</b> <?php echo htmlspecialchars($p['location']); ?></p>
        <div class="price">$<?php echo number_format($p['price'],2); ?>/<?php echo htmlspecialchars($p['unit']); ?></div>
        <a class="btn" href="product_details.php?id=<?php echo $p['id']; ?>">View Product</a>
        <?php if (is_buyer()): ?>
            <a class="btn btn-light" href="add_to_cart.php?id=<?php echo $p['id']; ?>">Add to Cart</a>
        <?php endif; ?>
        <?php if (is_admin()): ?>
            <a class="btn btn-light" href="edit_product.php?id=<?php echo $p['id']; ?>">Edit</a>
            <a class="btn btn-danger" onclick="return confirm('Delete this product?')" href="delete_product.php?id=<?php echo $p['id']; ?>">Delete</a>
        <?php endif; ?>
    </div>
<?php endwhile; ?>
</div>
</section>
<?php include "footer.php"; ?>
<?php
require_once "config.php";
require_admin();
$message = "";

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $name = $_POST["name"];
    $price = $_POST["price"];
    $unit = $_POST["unit"];
    $quantity = $_POST["quantity"];
    $location = $_POST["location"];
    $description = $_POST["description"];
    $image_path = "";

    if (!empty($_FILES["image"]["name"])) {
        $filename = time() . "_" . basename($_FILES["image"]["name"]);
        $target = "uploads/" . $filename;
        move_uploaded_file($_FILES["image"]["tmp_name"], $target);
        $image_path = $target;
    }

    $created_by = $_SESSION["user_id"];
    $stmt = $conn->prepare("INSERT INTO products (name, price, unit, quantity, location, description, image, created_by) VALUES (?, ?, ?, ?, ?, ?, ?, ?)");
    $stmt->bind_param("sdsssssi", $name, $price, $unit, $quantity, $location, $description, $image_path, $created_by);

    if ($stmt->execute()) {
        header("Location: marketplace.php");
        exit();
    } else {
        $message = "Error adding product.";
    }
}
include "header.php";
?>
<section class="page-header"><h1>Add Produce</h1><p>Add product details and upload an image.</p></section>
<section class="section">
<form method="POST" enctype="multipart/form-data">
    <p style="color:red;"><?php echo $message; ?></p>
    <label>Product Name</label><input name="name" required>
    <label>Price</label><input type="number" step="0.01" name="price" required>
    <label>Unit</label><input name="unit" value="kg" required>
    <label>Quantity</label><input name="quantity" placeholder="Example: 100 kg" required>
    <label>Location</label><input name="location" required>
    <label>Description</label><textarea name="description" required></textarea>
    <label>Upload Image</label><input type="file" name="image" accept="image/*" required>
    <button class="btn" type="submit">Add Produce</button>
</form>
</section>
<?php include "footer.php"; ?>
<?php
require_once "config.php";
require_buyer();
$product_id = intval($_GET["id"] ?? 0);
$buyer_id = $_SESSION["user_id"];

$stmt = $conn->prepare("SELECT id, quantity FROM cart WHERE buyer_id=? AND product_id=?");
$stmt->bind_param("ii", $buyer_id, $product_id);
$stmt->execute();
$item = $stmt->get_result()->fetch_assoc();

if ($item) {
    $qty = $item["quantity"] + 1;
    $stmt = $conn->prepare("UPDATE cart SET quantity=? WHERE id=?");
    $stmt->bind_param("ii", $qty, $item["id"]);
} else {
    $stmt = $conn->prepare("INSERT INTO cart (buyer_id, product_id, quantity) VALUES (?, ?, 1)");
    $stmt->bind_param("ii", $buyer_id, $product_id);
}
$stmt->execute();
header("Location: cart.php");
exit();
?>
<?php require_once "config.php"; require_buyer(); include "header.php"; ?>
<section class="page-header"><h1>Shopping Cart</h1><p>Review products before checkout.</p></section>
<section class="section">
<table>
<tr><th>Product</th><th>Image</th><th>Qty</th><th>Price</th><th>Total</th><th>Action</th></tr>
<?php
$buyer_id = $_SESSION["user_id"];
$stmt = $conn->prepare("SELECT cart.id AS cart_id, cart.quantity AS qty, products.* FROM cart JOIN products ON cart.product_id=products.id WHERE cart.buyer_id=?");
$stmt->bind_param("i", $buyer_id);
$stmt->execute();
$result = $stmt->get_result();
$total = 0;

while ($item = $result->fetch_assoc()):
    $line = $item["price"] * $item["qty"];
    $total += $line;
?>
<tr>
    <td><?php echo htmlspecialchars($item["name"]); ?></td>
    <td><img src="<?php echo htmlspecialchars($item["image"]); ?>" style="width:100px;height:70px;object-fit:cover;border-radius:12px;"></td>
    <td><?php echo $item["qty"]; ?></td>
    <td>$<?php echo number_format($item["price"],2); ?></td>
    <td>$<?php echo number_format($line,2); ?></td>
    <td><a class="btn btn-danger" href="remove_cart.php?id=<?php echo $item['cart_id']; ?>">Remove</a></td>
</tr>
<?php endwhile; ?>
<tr><th colspan="4">Grand Total</th><th>$<?php echo number_format($total,2); ?></th><th></th></tr>
</table>
<br>
<a class="btn btn-light" href="marketplace.php">Continue Shopping</a>
<?php if ($total > 0): ?><a class="btn" href="checkout.php">Checkout</a><?php endif; ?>
</section>
<?php include "footer.php"; ?>
<?php
require_once "config.php";
require_buyer();

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $buyer_id = $_SESSION["user_id"];
    $buyer_name = $_POST["buyer_name"];
    $address = $_POST["address"];
    $payment = $_POST["payment_method"];

    $stmt = $conn->prepare("INSERT INTO orders (buyer_id, buyer_name, address, payment_method) VALUES (?, ?, ?, ?)");
    $stmt->bind_param("isss", $buyer_id, $buyer_name, $address, $payment);
    $stmt->execute();
    $order_id = $conn->insert_id;

    $cart = $conn->prepare("SELECT cart.quantity AS qty, products.* FROM cart JOIN products ON cart.product_id=products.id WHERE cart.buyer_id=?");
    $cart->bind_param("i", $buyer_id);
    $cart->execute();
    $result = $cart->get_result();

    while ($item = $result->fetch_assoc()) {
        $stmt = $conn->prepare("INSERT INTO order_items (order_id, product_id, product_name, price, quantity) VALUES (?, ?, ?, ?, ?)");
        $stmt->bind_param("iisdi", $order_id, $item["id"], $item["name"], $item["price"], $item["qty"]);
        $stmt->execute();
    }

    $stmt = $conn->prepare("DELETE FROM cart WHERE buyer_id=?");
    $stmt->bind_param("i", $buyer_id);
    $stmt->execute();

    header("Location: buyer_orders.php");
    exit();
}
include "header.php";
?>
<section class="page-header"><h1>Checkout</h1><p>Enter delivery details to place your order.</p></section>
<section class="section">
<form method="POST">
    <label>Full Name</label><input name="buyer_name" value="<?php echo htmlspecialchars($_SESSION['full_name']); ?>" required>
    <label>Delivery Address</label><textarea name="address" required></textarea>
    <label>Payment Method</label>
    <select name="payment_method">
        <option>Cash on Delivery</option>
        <option>Card Payment</option>
        <option>Digital Wallet</option>
    </select>
    <button class="btn" type="submit">Place Order</button>
</form>
</section>
<?php include "footer.php"; ?>
<?php require_once "config.php"; require_buyer(); include "header.php"; ?>
<section class="page-header"><h1>Buyer Dashboard</h1><p>Explore products, add items to cart and track your purchases.</p></section>
<section class="section">
<div class="stats">
    <div class="stat"><p>Items in Cart</p><h3><?php echo cart_count($conn); ?></h3></div>
    <div class="stat"><p>Total Orders</p><h3><?php $r=$conn->prepare("SELECT COUNT(*) AS c FROM orders WHERE buyer_id=?"); $r->bind_param("i", $_SESSION['user_id']); $r->execute(); echo $r->get_result()->fetch_assoc()['c']; ?></h3></div>
    <div class="stat"><p>Account</p><h3>Active</h3></div>
</div>
<br>
<a class="btn" href="marketplace.php">Explore Marketplace</a>
<a class="btn btn-light" href="cart.php">View Cart</a>
<a class="btn btn-light" href="buyer_orders.php">Track My Orders</a>
</section>
<?php include "footer.php"; ?>
<?php require_once "config.php"; require_admin(); include "header.php"; ?>
<section class="page-header"><h1>Farmer/Admin Dashboard</h1><p>Manage produce, orders and reports.</p></section>
<section class="section">
<div class="stats">
    <div class="stat"><p>Total Products</p><h3><?php echo $conn->query("SELECT COUNT(*) AS c FROM products")->fetch_assoc()['c']; ?></h3></div>
    <div class="stat"><p>New Orders</p><h3><?php echo order_count($conn); ?></h3></div>
    <div class="stat"><p>Total Orders</p><h3><?php echo $conn->query("SELECT COUNT(*) AS c FROM orders")->fetch_assoc()['c']; ?></h3></div>
    <div class="stat"><p>Users</p><h3><?php echo $conn->query("SELECT COUNT(*) AS c FROM users")->fetch_assoc()['c']; ?></h3></div>
</div>
<br>
<a class="btn" href="add_product.php">Add Produce</a>
<a class="btn btn-light" href="orders.php">View Orders</a>
<a class="btn btn-light" href="marketplace.php">Manage Marketplace</a>
</section>
<?php include "footer.php"; ?>
