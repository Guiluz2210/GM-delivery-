/app
├── /static
├── /templates
├── app.py
├── models.py
└── config.py
from flask import Flask, render_template, request, redirect, url_for, flash, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
from flask_bcrypt import Bcrypt
import stripe

app = Flask(__name__)
app.secret_key = 'mysecretkey'
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://username:password@localhost/z_delivery'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
login_manager = LoginManager(app)
login_manager.login_view = "login"

# Stripe API Key (você precisa configurar isso)
stripe.api_key = "your_stripe_secret_key"

# Modelos de banco de dados
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(150), unique=True, nullable=False)
    email = db.Column(db.String(150), unique=True, nullable=False)
    password = db.Column(db.String(150), nullable=False)
    orders = db.relationship('Order', backref='user', lazy=True)

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(300), nullable=True)
    price = db.Column(db.Float, nullable=False)
    stock = db.Column(db.Integer, nullable=False)

class Order(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    product_id = db.Column(db.Integer, db.ForeignKey('product.id'), nullable=False)
    quantity = db.Column(db.Integer, nullable=False)
    status = db.Column(db.String(20), default='pendente')

class Coupon(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String(20), unique=True, nullable=False)
    discount = db.Column(db.Float, nullable=False)

# Criar as tabelas no banco de dados
db.create_all()

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# Rotas
@app.route('/')
def home():
    products = Product.query.all()
    return render_template('index.html', products=products)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']
        user = User.query.filter_by(email=email).first()
        if user and bcrypt.check_password_hash(user.password, password):
            login_user(user)
            return redirect(url_for('home'))
        else:
            flash('Email ou senha inválidos.', 'danger')
    return render_template('login.html')

@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('home'))

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        email = request.form['email']
        password = bcrypt.generate_password_hash(request.form['password']).decode('utf-8')
        new_user = User(username=username, email=email, password=password)
        db.session.add(new_user)
        db.session.commit()
        flash('Conta criada com sucesso!', 'success')
        return redirect(url_for('login'))
    return render_template('register.html')

@app.route('/add_product', methods=['POST'])
def add_product():
    if request.method == 'POST':
        name = request.form['name']
        description = request.form['description']
        price = float(request.form['price'])
        stock = int(request.form['stock'])
        new_product = Product(name=name, description=description, price=price, stock=stock)
        db.session.add(new_product)
        db.session.commit()
        flash('Produto adicionado com sucesso!', 'success')
        return redirect(url_for('home'))

@app.route('/checkout', methods=['GET', 'POST'])
@login_required
def checkout():
    cart = request.form.getlist('cart')  # Lista de IDs de produtos no carrinho
    total_amount = 0
    for product_id in cart:
        product = Product.query.get(int(product_id))
        total_amount += product.price
    if request.method == 'POST':
        coupon_code = request.form['coupon']
        coupon = Coupon.query.filter_by(code=coupon_code).first()
        if coupon:
            total_amount *= (1 - coupon.discount / 100)  # Aplica o desconto do cupom
        # Processar pagamento com Stripe
        token = request.form['stripeToken']
        try:
            charge = stripe.Charge.create(
                amount=int(total_amount * 100),  # Valor em centavos
                currency='brl',
                source=token,
                description='Pagamento de pedido'
            )
            flash('Pagamento realizado com sucesso!', 'success')
            return redirect(url_for('home'))
        except stripe.error.StripeError:
            flash('Erro no pagamento. Tente novamente.', 'danger')
    return render_template('checkout.html', total=total_amount)

# Rodar a aplicação
if __name__ == '__main__':
    app.run(debug=True)
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Zé Delivery</title>
</head>
<body>
    <h1>Produtos</h1>
    <ul>
        {% for product in products %}
        <li>{{ product.name }} - R${{ product.price }} 
            <form action="{{ url_for('add_to_cart', product_id=product.id) }}" method="POST">
                <button type="submit">Adicionar ao Carrinho</button>
            </form>
        </li>
        {% endfor %}
    </ul>
</body>
</html>
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <title>Checkout</title>
</head>
<body>
    <h1>Checkout</h1>
    <form action="{{ url_for('checkout') }}" method="POST">
        <label for="coupon">Cupom de Desconto</label>
        <input type="text" name="coupon" placeholder="Digite seu cupom">
        <script src="https://js.stripe.com/v3/"></script>
        <button type="submit">Pagar - R${{ total }}</button>
    </form>
</body>
</html>
