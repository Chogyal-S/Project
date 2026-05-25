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
