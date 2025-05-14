# Rewriting the full banking system using text files with the enhancements described.

import os
import datetime
import random

# File names
customers_file = "customers.txt"
users_file = "users.txt"
accounts_file = "accounts.txt"
transactions_file = "transactions.txt"
admins_file = "admins.txt"

# Get current timestamp
def get_time():
    return datetime.datetime.now().strftime("%d-%m-%Y %H:%M:%S")

# ---------------- File Operations ----------------

def load_data(file):
    if not os.path.exists(file):
        return []
    with open(file, "r") as f:
        return [line.strip() for line in f.readlines()]

def save_data(file, lines):
    with open(file, "w") as f:
        f.write("\n".join(lines) + ("\n" if lines else ""))

# ---------------- Authentication ----------------

def authenticate(role):
    attempts = 0
    file = admins_file if role == "admin" else users_file
    while attempts < 3:
        username = input("Username: ")
        password = input("Password: ")
        for line in load_data(file):
            parts = line.split(",")
            if role == "admin" and len(parts) == 2 and parts[0] == username and parts[1] == password:
                return True, username
            elif role == "user" and len(parts) == 3 and parts[1] == username and parts[2] == password:
                return True, parts[0]
        print("Invalid credentials.")
        attempts += 1
    print("Too many failed attempts. Exiting.")
    exit()

# ---------------- Account Creation ----------------

def setup_admin():
    print("=== Initial Admin Setup ===")
    username = input("Admin username: ")
    while True:
        password = input("Admin password: ")
        if len(password) >= 6:
            break
        print("Password must be at least 6 characters.")
    save_data(admins_file, [f"{username},{password}"])
    print("Admin account created.")

def get_new_account_number():
    # Generate a unique 4-digit account number (1000–9999)
    return str(random.randint(1000, 9999))

def create_account():
    customer_id = f"C{random.randint(1000, 9999)}"
    while True:
        name = input("Enter customer name: ").strip().title()
        if all(x.isalpha() or x.isspace() for x in name):
            break
        print("Invalid name. Only letters and spaces allowed.")
    while True:
        username = input("Set username for customer: ")
        password = input("Set password (min 6 characters): ")
        if len(password) >= 6:
            break
        print("Password too short.")

    try:
        balance = float(input("Enter initial balance: "))
        if balance < 0:
            print("Initial balance must be non-negative.")
            return
    except ValueError:
        print("Invalid amount.")
        return

    acc_no = get_new_account_number()
    # Save to files
    with open(customers_file, "a") as f: f.write(f"{customer_id},{name}\n")
    with open(users_file, "a") as f: f.write(f"{customer_id},{username},{password}\n")
    with open(accounts_file, "a") as f: f.write(f"{acc_no},{customer_id},{balance}\n")
    with open(transactions_file, "a") as f: f.write(f"{acc_no},Deposit,{balance},{get_time()}\n")
    print(f"Account created for {name}. Account Number: {acc_no}")

# ---------------- Banking Features ----------------

def deposit_money(acc_no):
    try:
        amount = float(input("Amount to deposit: "))
        if amount < 0:
            print("Error: Deposit amount cannot be negative.")
            return
        if amount == 0:
            print("Error: Deposit amount must be positive.")
            return
    except ValueError:
        print("Invalid amount.")
        return
    updated = False
    lines = load_data(accounts_file)
    new_lines = []
    for line in lines:
        acc, cust_id, bal = line.split(",")
        if acc == acc_no:
            bal = str(float(bal) + amount)
            new_lines.append(f"{acc},{cust_id},{bal}")
            updated = True
        else:
            new_lines.append(line)
    if updated:
        save_data(accounts_file, new_lines)
        with open(transactions_file, "a") as f:
            f.write(f"{acc_no},Deposit,{amount},{get_time()}\n")
        print("Deposit successful.")
    else:
        print("Account not found.")

def withdraw_money(acc_no):
    try:
        amount = float(input("Amount to withdraw: "))
        if amount <= 0:
            print("Amount must be positive.")
            return
    except ValueError:
        print("Invalid input.")
        return
    updated = False
    lines = load_data(accounts_file)
    new_lines = []
    for line in lines:
        acc, cust_id, bal = line.split(",")
        if acc == acc_no:
            bal = float(bal)
            if bal >= amount:
                bal -= amount
                new_lines.append(f"{acc},{cust_id},{bal}")
                updated = True
            else:
                print("Insufficient balance.")
                return
        else:
            new_lines.append(line)
    if updated:
        save_data(accounts_file, new_lines)
        with open(transactions_file, "a") as f:
            f.write(f"{acc_no},Withdrawal,{amount},{get_time()}\n")
        print("Withdrawal successful.")
    else:
        print("Account not found.")

def check_balance(acc_no):
    for line in load_data(accounts_file):
        acc, cust_id, bal = line.split(",")
        if acc == acc_no:
            print(f"Balance: Rs.{bal}")
            return
    print("Account not found.")

def transaction_history(acc_no):
    print("=== Last 5 Transactions ===")
    transactions = [line for line in load_data(transactions_file) if line.startswith(acc_no)]
    if not transactions:
        print("No transactions.")
        return
    for line in transactions[-5:]:
        _, t_type, amount, time = line.split(",")
        print(f"{time} | {t_type:<12} | Rs.{amount}")

def transfer_money(from_acc):
    to_acc = input("Recipient account number: ")
    if from_acc == to_acc:
        print("Sender and receiver cannot be the same.")
        return
    try:
        amount = float(input("Amount to transfer: "))
        if amount <= 0:
            print("Amount must be positive.")
            return
    except ValueError:
        print("Invalid input.")
        return

    acc_data = load_data(accounts_file)
    updated_lines = []
    from_found = to_found = False
    for line in acc_data:
        acc, cid, bal = line.split(",")
        if acc == from_acc:
            bal = float(bal)
            if bal < amount:
                print("Insufficient balance.")
                return
            bal -= amount
            from_found = True
        elif acc == to_acc:
            bal = float(bal) + amount
            to_found = True
        else:
            updated_lines.append(line)
            continue
        updated_lines.append(f"{acc},{cid},{bal}")
    if from_found and to_found:
        save_data(accounts_file, updated_lines)
        time = get_time()
        with open(transactions_file, "a") as f:
            f.write(f"{from_acc},TransferOut,{amount},{time}\n")
            f.write(f"{to_acc},TransferIn,{amount},{time}\n")
        print("Transfer successful.")
    else:
        print("One of the accounts was not found.")

def total_transactions():
    count = len(load_data(transactions_file))
    print(f"Total Transactions: {count}")

def display_customer_list():
    lines = load_data(customers_file)
    if not lines:
        print("No customers found.")
        return
    print("=== Customer List ===")
    for line in lines:
        cid, name = line.split(",")
        print(f"{cid}: {name}")

def delete_customer():
    cid = input("Enter customer ID to delete: ")
    confirm = input("Are you sure? (Y/N): ").strip().upper()
    if confirm != "Y":
        return
    # Remove from all files
    def keep_line(file, idx, match):
        return [l for l in load_data(file) if l.split(",")[idx] != match]

    save_data(customers_file, keep_line(customers_file, 0, cid))
    save_data(users_file, keep_line(users_file, 0, cid))
    accounts = [l for l in load_data(accounts_file) if l.split(",")[1] != cid]
    removed_accs = [l.split(",")[0] for l in load_data(accounts_file) if l.split(",")[1] == cid]
    save_data(accounts_file, accounts)
    transactions = [l for l in load_data(transactions_file) if l.split(",")[0] not in removed_accs]
    save_data(transactions_file, transactions)
    print("Customer account deleted.")

# ---------------- Menus ----------------

def admin_menu(admin):
    while True:
        print(f"\n=== Admin Menu ({admin}) ===")
        print("1. Create Account\n2. Deposit\n3. Withdraw\n4. Check Balance")
        print("5. View Transactions\n6. Transfer Money\n7. Total Transactions")
        print("8. Customer List\n9. Delete Customer\n10. Logout")
        choice = input("Enter choice: ")
        if choice == "1":
            create_account()
        elif choice == "2":
            deposit_money(input("Enter account number: "))
        elif choice == "3":
            withdraw_money(input("Enter account number: "))
        elif choice == "4":
            check_balance(input("Enter account number: "))
        elif choice == "5":
            transaction_history(input("Enter account number: "))
        elif choice == "6":
            transfer_money(input("Enter sender's account number: "))
        elif choice == "7":
            total_transactions()
        elif choice == "8":
            display_customer_list()
        elif choice == "9":
            delete_customer()
        elif choice == "10":
            break
        else:
            print("Invalid option.")

def user_menu(account_number):
    while True:
        print(f"\n=== User Menu ({account_number}) ===")
        print("1. Deposit\n2. Withdraw\n3. Check Balance")
        print("4. View Transactions\n5. Transfer Money\n6. Logout")
        choice = input("Enter choice: ")
        if choice == "1":
            deposit_money(account_number)
        elif choice == "2":
            withdraw_money(account_number)
        elif choice == "3":
            check_balance(account_number)
        elif choice == "4":
            transaction_history(account_number)
        elif choice == "5":
            transfer_money(account_number)
        elif choice == "6":
            break
        else:
            print("Invalid option.")

# ---------------- Main ----------------

def main():
    if not os.path.exists(admins_file) or not load_data(admins_file):
        setup_admin()
    while True:
        print("\nLogin as:\n1. Admin\n2. User\n3. Exit")
        choice = input("Choice: ")
        if choice == "1":
            success, admin = authenticate("admin")
            if success:
                admin_menu(admin)
        elif choice == "2":
            success, acc_no = authenticate("user")
            if success:
                user_menu(acc_no)
        elif choice == "3":
            print("Thank you for using the Banking App.")
            break
        else:
            print("Invalid choice.")

main()