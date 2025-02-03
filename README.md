# ระบบการจัดการร้านคอมพิ้วเตอร์ด้วยภาษาC++ เเละ Mysql
#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <unordered_map>
#include <iomanip>
#include <algorithm>
#include <ctime>
#include <sstream>
#include <limits>

using namespace std;

// โครงสร้าง User
struct User {
    string username;
    string password;
    string role;
    string email;
    string name;
};

// โครงสร้าง Product
struct Product {
    string name;
    string brand;
    string model;
    string code;
    double price;
    int quantity;
    string warrantyPeriod;      // ระยะเวลารับประกัน
    string warrantyConditions;  // เงื่อนไขการรับประกัน
};

// โครงสร้าง RepairRecord สำหรับเก็บข้อมูลการซ่อมสินค้า
struct RepairRecord {
    string code;
    string repairReason;
    string repairDate;
    string repairStatus;  // สถานะการซ่อม
};

// โครงสร้าง Customer
struct Customer {
    string username;      // ชื่อผู้ใช้ (username)
    string name;          // ชื่อจริง
    string address;       // ที่อยู่
    string contactNumber; // เบอร์ติดต่อ
};

// โครงสร้าง PurchaseRecord สำหรับเก็บประวัติการซื้อสินค้า
struct PurchaseRecord {
    string username;
    string productCode;
    string productName;
    int quantity;
    double totalPrice;
    string purchaseDate;
    string warrantyPeriod;
    string warrantyConditions;
    string salespersonUsername; // เพิ่มชื่อผู้ขาย
    string customerName;        // เพิ่มชื่อลูกค้า
};

// โครงสร้าง InventoryChangeRecord สำหรับเก็บประวัติการเข้า-ออกของสินค้า
struct InventoryChangeRecord {
    string productCode;
    string productName;
    int quantityChanged;
    string changeType; // "Added" or "Removed"
    string date;
};

// การประกาศฟังก์ชันที่ใช้ในโปรแกรม (โปรโตไทป์ของฟังก์ชัน)
// ฟังก์ชันเกี่ยวกับผู้ใช้และสิทธิ์การเข้าถึง
void assignPermissions(const string& username, const string& role);
void saveUserToFile(const User& user);
void loadUsersFromFile();
bool login();
void registerUser();
void mainMenu();

// ฟังก์ชันเกี่ยวกับ User Management
void showUserManagementMenu();
void viewAllUsers();
void editUserRole();

// ฟังก์ชันเกี่ยวกับสินค้าคงคลัง
void saveAllProductsToFile();
void loadInventoryFromFile();
void addProduct();
void removeProduct();
void editProduct();
void checkStockLevel();
void lowStockAlert();
void showInventorySubmenu();
void recordInventoryChange(const string& productCode, const string& productName, int quantityChanged, const string& changeType);
void loadInventoryChangesFromFile();
void showInventoryChangeHistory();

// ฟังก์ชันเกี่ยวกับการขาย
void sellProduct(const string& username);
void generateReceipt(const string& username); // แก้ไขเพิ่มพารามิเตอร์ username
void searchProduct();
void loadAllSalesRecordsFromFile();
void showSalesSubmenu(const string& username);
void viewOrderHistory(); // เพิ่มฟังก์ชันนี้

// ฟังก์ชันเกี่ยวกับการรับประกันและการซ่อม
void recordWarrantyDetails();
void recordRepair();
void viewRepairHistory();
void loadRepairHistoryFromFile();
void showWarrantySubmenu();
void showRepairReport();
void editRepairRecord(); // เพิ่มฟังก์ชันแก้ไขข้อมูลการซ่อม

// ฟังก์ชันเกี่ยวกับลูกค้า
void recordCustomerInfo(const string& username);
void loadCustomersFromFile();
void viewCustomerInfo(const string& username);
void editCustomerInfo(const string& username);
void purchaseProducts(const string& username);
void loadPurchaseHistoryFromFile();
void viewPurchaseHistory(const string& username);
void showCustomerMenu(const string& username);
void showCustomerManagementMenu();
void viewAllCustomers();

// ฟังก์ชันเกี่ยวกับรายงาน
void showSalesReport();
void showSalesReportByPeriod();
void showInventoryReport();
void showReportingSystemMenu();

// ฟังก์ชันอื่น ๆ
void showUserMenu(const string& username);

// ข้อมูลผู้ใช้และสิทธิ์การเข้าถึงสำหรับแต่ละประเภท
unordered_map<string, User> users; // เปลี่ยนเป็นเก็บข้อมูล User
unordered_map<string, vector<string>> accessPermissions;
unordered_map<string, Product> inventory;
unordered_map<string, vector<RepairRecord>> repairHistory;  // ประวัติการซ่อมสินค้า
unordered_map<string, Customer> customers; // ข้อมูลลูกค้า
unordered_map<string, vector<PurchaseRecord>> purchaseHistory; // ประวัติการซื้อสินค้าของลูกค้าแต่ละคน
vector<PurchaseRecord> allSalesRecords; // ประวัติการขายทั้งหมด
vector<InventoryChangeRecord> inventoryChanges; // ประวัติการเข้า-ออกของสินค้า

const int lowStockThreshold = 5;  // กำหนดระดับสินค้าที่ต้องแจ้งเตือน
const int maxStockLimit = 1000;   // กำหนดจำนวนสินค้าสูงสุดในคลังสินค้า

// กำหนดสิทธิ์การเข้าถึงสำหรับผู้ใช้
void assignPermissions(const string& username, const string& role) {
    if (role == "Administrator") {
        accessPermissions[username] = {"Inventory Management", "Sales System", "Warranty and Repair System",
                                       "Customer Management", "Reporting System", "User Management"};
    } else if (role == "Salesperson") {
        accessPermissions[username] = {"Sales System", "Customer Management", "Reporting System", "Warranty and Repair System"};
    } else if (role == "Warehouse Staff") {
        accessPermissions[username] = {"Inventory Management", "Reporting System", "Warranty and Repair System"};
    } else if (role == "Customer") {
        // ลูกค้าไม่ต้องการสิทธิ์การเข้าถึงเมนูหลัก
        accessPermissions[username] = {};
    }
}

// ฟังก์ชันบันทึกผู้ใช้
void saveUserToFile(const User& user) {
    ofstream file("users.txt", ios::app);
    file << user.username << " " << user.password << " " << user.role << " " << user.email << " " << user.name << endl;
    file.close();
}

// ฟังก์ชันโหลดข้อมูลผู้ใช้
void loadUsersFromFile() {
    ifstream file("users.txt");
    if (!file) {
        cout << "Error: Could not open users.txt file.\n";
        return;
    }

    string line;
    while (getline(file, line)) {
        if (line.empty()) continue; // ข้ามบรรทัดว่าง

        istringstream iss(line);
        string username, password, role, email, name;
        iss >> username >> password >> role >> email;
        getline(iss, name); // อ่านชื่อที่เหลืออยู่ในบรรทัด

        if (!username.empty() && !password.empty() && !role.empty()) {
            User user = {username, password, role, email, name};
            users[username] = user;
            assignPermissions(username, role);
        } else {
            cout << "Error: Incomplete user data in users.txt. Line skipped.\n";
        }
    }
    file.close();
}

// ฟังก์ชันสำหรับการ Login
bool login() {
    string username, password;
    cout << "Enter username: ";
    cin >> username;
    cout << "Enter password: ";
    cin >> password;

    if (users.find(username) != users.end() && users[username].password == password) {
        cout << "Login successful! Welcome, " << users[username].name << " (" << users[username].role << ")" << endl;
        if (users[username].role == "Customer") {
            showCustomerMenu(username);
        } else {
            showUserMenu(username);
        }
        return true;
    } else {
        cout << "Incorrect username or password." << endl;
        return false;
    }
}

// ฟังก์ชันลงทะเบียน (Register) พร้อมการตรวจสอบอินพุต
void registerUser() {
    string username, password, email, role, name;
    int userType;

    cout << "Choose account type:\n1. Salesperson\n2. Warehouse Staff\n3. Customer\n";

    // ตรวจสอบการใส่ค่าให้ถูกต้อง
    while (true) {
        cout << "Enter your choice: ";
        cin >> userType;
        if (cin.fail() || userType < 1 || userType > 3) {
            cout << "Invalid choice. Please enter a number between 1 and 3.\n";
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        } else {
            break;
        }
    }
    cin.ignore();

    if (userType == 1)
        role = "Salesperson";
    else if (userType == 2)
        role = "Warehouse Staff";
    else if (userType == 3)
        role = "Customer";

    // ตรวจสอบว่าชื่อผู้ใช้ซ้ำหรือไม่
    while (true) {
        cout << "Enter username: ";
        cin >> username;
        if (users.find(username) != users.end()) {  // ตรวจสอบว่าชื่อผู้ใช้มีอยู่แล้วหรือไม่
            cout << "Username already exists. Please enter a different username.\n";
        } else {
            break;
        }
    }

    cout << "Enter password: ";
    cin >> password;
    cout << "Enter email: ";
    cin >> email;
    cout << "Enter full name: ";
    cin.ignore();
    getline(cin, name);

    User user = {username, password, role, email, name};
    saveUserToFile(user);
    users[username] = user;

    assignPermissions(username, role);
    cout << "Registration successful! Access permissions have been assigned.\n";

    if (role == "Customer") {
        recordCustomerInfo(username);
    }
}

// เมนูหลัก พร้อมเมนู 'Back' ในทุกเมนู
void mainMenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Main Menu ===\n";
        cout << "1. Login\n";
        cout << "2. Register\n";
        cout << "0. Exit\n";
        cout << "Enter your choice: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 2) {
                cout << "Invalid choice. Please enter 0, 1, or 2: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                login();
                break;
            case 2:
                registerUser();
                break;
            case 0:
                cout << "Exiting the system. Goodbye!\n";
                running = false;
                break;
        }
    }
}

// ฟังก์ชันแสดงเมนูสำหรับการจัดการผู้ใช้ พร้อมเมนู 'Back'
void showUserManagementMenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== User Management ===\n";
        cout << "1. View All Users\n";
        cout << "2. Edit User Role\n";
        cout << "0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 2) {
                cout << "Invalid choice. Please enter 0, 1, or 2: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                viewAllUsers();
                break;
            case 2:
                editUserRole();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันดูข้อมูลผู้ใช้ทั้งหมด
void viewAllUsers() {
    cout << "\n--- All Users ---\n";
    cout << left << setw(15) << "Username" << setw(25) << "Name" << setw(15) << "Role" << endl;
    cout << string(55, '-') << endl;
    for (const auto& pair : users) {
        const User& user = pair.second;
        cout << left << setw(15) << user.username << setw(25) << user.name << setw(15) << user.role << endl;
    }
}

// ฟังก์ชันแก้ไขระดับการเข้าถึงของผู้ใช้ พร้อมการตรวจสอบอินพุต
void editUserRole() {
    string username;
    cout << "Enter the username of the user to edit (or '0' to cancel): ";
    cin >> username;

    if (username == "0") {
        return;
    }

    if (users.find(username) != users.end()) {
        User& user = users[username];
        cout << "Current Role: " << user.role << endl;
        cout << "Available Roles:\n1. Administrator\n2. Salesperson\n3. Warehouse Staff\n4. Customer\n";

        int roleChoice;
        while (true) {
            cout << "Enter new role (1-4): ";
            cin >> roleChoice;

            if (cin.fail() || roleChoice < 1 || roleChoice > 4) {
                cout << "Invalid choice. Please enter a number between 1 and 4.\n";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        string newRole;
        if (roleChoice == 1)
            newRole = "Administrator";
        else if (roleChoice == 2)
            newRole = "Salesperson";
        else if (roleChoice == 3)
            newRole = "Warehouse Staff";
        else if (roleChoice == 4)
            newRole = "Customer";

        user.role = newRole;
        assignPermissions(username, newRole);

        // อัปเดตไฟล์ users.txt
        ofstream file("users.txt", ios::trunc);
        for (const auto& pair : users) {
            const User& u = pair.second;
            file << u.username << " " << u.password << " " << u.role << " " << u.email << " " << u.name << endl;
        }
        file.close();

        cout << "User role updated successfully.\n";
    } else {
        cout << "User not found.\n";
    }
}

// ฟังก์ชันบันทึกสินค้าทั้งหมดในไฟล์
void saveAllProductsToFile() {
    ofstream file("inventory.txt", ios::trunc);
    for (const auto& item : inventory) {
        const Product& product = item.second;
        file << product.name << " " << product.brand << " " << product.model << " "
             << product.code << " " << product.price << " " << product.quantity << " "
             << product.warrantyPeriod << " " << product.warrantyConditions << endl;
    }
    file.close();
}

// ฟังก์ชันโหลดสินค้าจากไฟล์
void loadInventoryFromFile() {
    ifstream file("inventory.txt");
    if (file) {
        Product product;
        while (file >> product.name >> product.brand >> product.model >> product.code >> product.price >> product.quantity >> product.warrantyPeriod >> product.warrantyConditions) {
            // แปลงขีดล่างเป็นช่องว่าง
            replace(product.name.begin(), product.name.end(), '_', ' ');
            replace(product.brand.begin(), product.brand.end(), '_', ' ');
            replace(product.model.begin(), product.model.end(), '_', ' ');
            replace(product.warrantyPeriod.begin(), product.warrantyPeriod.end(), '_', ' ');
            replace(product.warrantyConditions.begin(), product.warrantyConditions.end(), '_', ' ');
            inventory[product.code] = product;
        }
        file.close();
    }
}

// ฟังก์ชันบันทึกประวัติการเข้า-ออกของสินค้า
void recordInventoryChange(const string& productCode, const string& productName, int quantityChanged, const string& changeType) {
    time_t now = time(0);
    tm* ltm = localtime(&now);
    char dateStr[20];
    strftime(dateStr, sizeof(dateStr), "%Y-%m-%d", ltm);
    string date(dateStr);

    InventoryChangeRecord record = {productCode, productName, quantityChanged, changeType, date};
    inventoryChanges.push_back(record);

    // บันทึกลงไฟล์
    ofstream file("inventory_changes.txt", ios::app);
    file << productCode << "\n" << productName << "\n" << quantityChanged << "\n" << changeType << "\n" << date << endl;
    file.close();
}

// ฟังก์ชันโหลดประวัติการเข้า-ออกของสินค้าจากไฟล์
void loadInventoryChangesFromFile() {
    ifstream file("inventory_changes.txt");
    if (file) {
        string productCode, productName, changeType, date;
        int quantityChanged;
        while (getline(file, productCode)) {
            getline(file, productName);
            file >> quantityChanged;
            file.ignore(); // ข้าม newline
            getline(file, changeType);
            getline(file, date);

            InventoryChangeRecord record = {productCode, productName, quantityChanged, changeType, date};
            inventoryChanges.push_back(record);
        }
        file.close();
    }
}

// ฟังก์ชันเพิ่มสินค้า พร้อมการตรวจสอบจำนวนสินค้า
void addProduct() {
    Product product;
    cout << "Enter Product Name (or '0' to cancel): ";
    cin >> product.name;
    if (product.name == "0") {
        return;
    }
    cout << "Enter Brand: ";
    cin >> product.brand;
    cout << "Enter Model: ";
    cin >> product.model;
    cout << "Enter Product Code (SKU): ";
    cin >> product.code;

    if (inventory.find(product.code) != inventory.end()) {
        cout << "Error: Product Code already exists. Please use a unique code.\n";
        return;
    }

    // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับราคา
    while (true) {
        cout << "Enter Price: ";
        cin >> product.price;
        if (cin.fail() || product.price < 0) {
            cout << "Invalid price. Please enter a positive number.\n";
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        } else {
            break;
        }
    }

    // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับจำนวนสินค้า
    int inputQuantity;
    while (true) {
        cout << "Enter Quantity (1 - " << maxStockLimit << "): ";
        cin >> inputQuantity;
        if (cin.fail() || inputQuantity < 1 || inputQuantity > maxStockLimit) {
            cout << "Invalid quantity. Please enter a number between 1 and " << maxStockLimit << ".\n";
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        } else {
            product.quantity = inputQuantity;
            break;
        }
    }

    cin.ignore();  // Clear the input buffer
    cout << "Enter Warranty Period: ";
    getline(cin, product.warrantyPeriod);
    cout << "Enter Warranty Conditions: ";
    getline(cin, product.warrantyConditions);

    inventory[product.code] = product;
    saveAllProductsToFile();
    cout << "Product added successfully!\n";

    // บันทึกประวัติการเข้า-ออกของสินค้า
    recordInventoryChange(product.code, product.name, product.quantity, "Added");
}

// ฟังก์ชันลบสินค้า พร้อมเมนู 'Back'
void removeProduct() {
    string code;
    cout << "Enter Product Code to remove (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) != inventory.end()) {
        int quantityRemoved = inventory[code].quantity;
        string productName = inventory[code].name;
        inventory.erase(code);
        saveAllProductsToFile();
        cout << "Product removed successfully!\n";

        // บันทึกประวัติการเข้า-ออกของสินค้า
        recordInventoryChange(code, productName, quantityRemoved, "Removed");
    } else {
        cout << "Product not found.\n";
    }
}

// ฟังก์ชันแก้ไขสินค้า พร้อมการตรวจสอบจำนวนสินค้า
void editProduct() {
    string code;
    cout << "Enter Product Code to edit (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) != inventory.end()) {
        Product& product = inventory[code];
        cout << "Editing Product: " << product.name << endl;

        cout << "Enter new Name (current: " << product.name << ", or '0' to keep): ";
        string input;
        cin >> input;
        if (input != "0") {
            product.name = input;
        }

        cout << "Enter new Brand (current: " << product.brand << ", or '0' to keep): ";
        cin >> input;
        if (input != "0") {
            product.brand = input;
        }

        cout << "Enter new Model (current: " << product.model << ", or '0' to keep): ";
        cin >> input;
        if (input != "0") {
            product.model = input;
        }

        // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับราคา
        double newPrice;
        while (true) {
            cout << "Enter new Price (current: " << product.price << ", or '-1' to keep): ";
            cin >> newPrice;
            if (cin.fail() || (newPrice < 0 && newPrice != -1)) {
                cout << "Invalid price. Please enter a positive number or '-1' to keep current.\n";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                if (newPrice != -1) {
                    product.price = newPrice;
                }
                break;
            }
        }

        // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับจำนวนสินค้า
        int newQuantity;
        while (true) {
            cout << "Enter new Quantity (current: " << product.quantity << ", Max " << maxStockLimit << ", or '-1' to keep): ";
            cin >> newQuantity;
            if (cin.fail() || (newQuantity != -1 && (newQuantity < 0 || newQuantity > maxStockLimit))) {
                cout << "Invalid quantity. Please enter a number between 0 and " << maxStockLimit << ", or '-1' to keep current.\n";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                if (newQuantity != -1) {
                    product.quantity = newQuantity;
                }
                break;
            }
        }

        cin.ignore();  // Clear the input buffer
        cout << "Enter new Warranty Period (current: " << product.warrantyPeriod << ", or '0' to keep): ";
        getline(cin, input);
        if (input != "0") {
            product.warrantyPeriod = input;
        }

        cout << "Enter new Warranty Conditions (current: " << product.warrantyConditions << ", or '0' to keep): ";
        getline(cin, input);
        if (input != "0") {
            product.warrantyConditions = input;
        }

        saveAllProductsToFile();
        cout << "Product updated successfully!\n";
    } else {
        cout << "Product not found.\n";
    }
}

// ฟังก์ชันตรวจสอบระดับสต็อก
void checkStockLevel() {
    cout << "\n--- Current Inventory ---\n";
    cout << left << setw(15) << "Product" << setw(10) << "Code" << setw(10) << "Quantity" << setw(10) << "Price" << setw(20) << "Warranty Period" << endl;
    cout << string(65, '-') << endl;
    for (const auto& item : inventory) {
        const Product& product = item.second;
        cout << left << setw(15) << product.name << setw(10) << product.code << setw(10) << product.quantity
             << setw(10) << product.price << setw(20) << product.warrantyPeriod << endl;
    }
}

// ฟังก์ชันแจ้งเตือนสต็อกต่ำ
void lowStockAlert() {
    cout << "\n--- Low Stock Alert ---\n";
    bool alert = false;
    for (const auto& item : inventory) {
        const Product& product = item.second;
        if (product.quantity < lowStockThreshold) {
            cout << "Product: " << product.name << " | Code: " << product.code
                 << " | Quantity: " << product.quantity << " (Low Stock)\n";
            alert = true;
        }
    }
    if (!alert) {
        cout << "All products are sufficiently stocked.\n";
    }
}

// ฟังก์ชันแสดงประวัติการเข้า-ออกของสินค้า
void showInventoryChangeHistory() {
    cout << "\n--- Inventory Change History ---\n";
    cout << left << setw(12) << "Date" << setw(15) << "Product" << setw(10) << "Code" << setw(8) << "Change" << setw(10) << "Quantity" << endl;
    cout << string(55, '-') << endl;
    for (const auto& record : inventoryChanges) {
        cout << left << setw(12) << record.date << setw(15) << record.productName << setw(10) << record.productCode
             << setw(8) << record.changeType << setw(10) << record.quantityChanged << endl;
    }
}

// ฟังก์ชันแสดงเมนูย่อยสำหรับ Inventory Management พร้อมเมนู 'Back'
void showInventorySubmenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Inventory Management Submenu ===\n";
        cout << "1. Add Product\n2. Remove Product\n3. Edit Product\n4. View Inventory\n";
        cout << "5. Low Stock Alert\n6. Inventory Change History\n0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 6) {
                cout << "Invalid choice. Please enter a number between 0 and 6: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                addProduct();
                break;
            case 2:
                removeProduct();
                break;
            case 3:
                editProduct();
                break;
            case 4:
                checkStockLevel();
                break;
            case 5:
                lowStockAlert();
                break;
            case 6:
                showInventoryChangeHistory();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันการขายสินค้า พร้อมการตรวจสอบจำนวนสินค้า
void sellProduct(const string& username) {
    string code, customerUsername;
    int quantity;
    cout << "Enter Customer Username (or '0' to cancel): ";
    cin >> customerUsername;

    if (customerUsername == "0") {
        return;
    }

    if (customers.find(customerUsername) == customers.end()) {
        cout << "Customer not found.\n";
        return;
    }

    cout << "Enter Product Code (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) != inventory.end()) {
        Product& product = inventory[code];

        // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับจำนวนสินค้า
        while (true) {
            cout << "Enter quantity to sell: ";
            cin >> quantity;
            if (cin.fail() || quantity <= 0) {
                cout << "Invalid quantity. Please enter a positive number.\n";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else if (quantity > product.quantity) {
                cout << "Insufficient stock. Available: " << product.quantity << endl;
                cout << "Please enter a new quantity or '0' to cancel.\n";
                continue;
            } else {
                break;
            }
        }

        if (quantity == 0) {
            return;
        }

        product.quantity -= quantity;
        double totalPrice = product.price * quantity;
        saveAllProductsToFile();
        cout << "\n--- Sale Receipt ---\n";
        cout << "Salesperson: " << users[username].name << "\n";
        cout << "Customer: " << customers[customerUsername].name << "\n";
        cout << "Product: " << product.name << "\n";
        cout << "Quantity: " << quantity << "\n";
        cout << "Total Price: $" << fixed << setprecision(2) << totalPrice << endl;
        cout << "---------------------\n";
        cout << "Sale recorded successfully!\n";

        // บันทึกประวัติการขาย
        time_t now = time(0);
        tm* ltm = localtime(&now);
        char dateStr[20];
        strftime(dateStr, sizeof(dateStr), "%Y-%m-%d", ltm);
        string saleDate(dateStr);

        PurchaseRecord record = {customerUsername, product.code, product.name, quantity, totalPrice, saleDate, product.warrantyPeriod, product.warrantyConditions, username, customers[customerUsername].name};
        allSalesRecords.push_back(record);

        // บันทึกลงไฟล์
        ofstream file("all_sales_records.txt", ios::app);
        file << customerUsername << "\n" << product.code << "\n" << product.name << "\n" << quantity << "\n" << totalPrice << "\n"
             << saleDate << "\n" << product.warrantyPeriod << "\n" << product.warrantyConditions << "\n"
             << username << "\n" << customers[customerUsername].name << endl;
        file.close();

        // บันทึกประวัติการเข้า-ออกของสินค้า
        recordInventoryChange(product.code, product.name, quantity, "Removed");
    } else {
        cout << "Product not found.\n";
    }
}

// ฟังก์ชันใบเสร็จ พร้อมการตรวจสอบจำนวนสินค้า
void generateReceipt(const string& username) { // เพิ่มพารามิเตอร์ username
    string code, customerUsername;
    int quantity;
    cout << "Enter Customer Username (or '0' to cancel): ";
    cin >> customerUsername;

    if (customerUsername == "0") {
        return;
    }

    if (customers.find(customerUsername) == customers.end()) {
        cout << "Customer not found.\n";
        return;
    }

    cout << "Enter Product Code (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) != inventory.end()) {
        Product& product = inventory[code];

        // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับจำนวนสินค้า
        while (true) {
            cout << "Enter quantity to purchase: ";
            cin >> quantity;
            if (cin.fail() || quantity <= 0) {
                cout << "Invalid quantity. Please enter a positive number.\n";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else if (quantity > product.quantity) {
                cout << "Insufficient stock. Available: " << product.quantity << endl;
                cout << "Please enter a new quantity or '0' to cancel.\n";
                continue;
            } else {
                break;
            }
        }

        if (quantity == 0) {
            return;
        }

        product.quantity -= quantity;
        double totalPrice = product.price * quantity;
        saveAllProductsToFile();
        cout << "\n--- Receipt ---\n";
        cout << "Salesperson: " << users[username].name << "\n";
        cout << "Customer: " << customers[customerUsername].name << "\n";
        cout << "Product: " << product.name << "\nQuantity: " << quantity
             << "\nTotal Price: $" << fixed << setprecision(2) << totalPrice << endl;
        cout << "----------------\n";

        // บันทึกประวัติการเข้า-ออกของสินค้า
        recordInventoryChange(product.code, product.name, quantity, "Removed");
    } else {
        cout << "Product not found.\n";
    }
}

// ฟังก์ชันค้นหาสินค้า
void searchProduct() {
    string searchTerm;
    cout << "Enter product name, code, or brand to search: ";
    cin >> searchTerm;

    bool found = false;
    cout << "\n--- Search Results ---\n";
    cout << left << setw(15) << "Product" << setw(10) << "Code" << setw(10) << "Price" << setw(10) << "Quantity" << endl;
    cout << string(45, '-') << endl;
    for (const auto& item : inventory) {
        const Product& product = item.second;
        if (product.name.find(searchTerm) != string::npos || product.code.find(searchTerm) != string::npos || product.brand.find(searchTerm) != string::npos) {
            cout << left << setw(15) << product.name << setw(10) << product.code << setw(10) << product.price << setw(10) << product.quantity << endl;
            found = true;
        }
    }
    if (!found) {
        cout << "Product not found.\n";
    }
}

// ฟังก์ชันโหลดประวัติการขายทั้งหมดจากไฟล์
void loadAllSalesRecordsFromFile() {
    ifstream file("all_sales_records.txt");
    if (file) {
        string username, code, productName, purchaseDate, warrantyPeriod, warrantyConditions, salespersonUsername, customerName;
        int quantity;
        double totalPrice;
        while (getline(file, username)) {
            getline(file, code);
            getline(file, productName);
            file >> quantity;
            file >> totalPrice;
            file.ignore(); // ข้าม newline
            getline(file, purchaseDate);
            getline(file, warrantyPeriod);
            getline(file, warrantyConditions);
            getline(file, salespersonUsername);
            getline(file, customerName);

            PurchaseRecord record = {username, code, productName, quantity, totalPrice, purchaseDate, warrantyPeriod, warrantyConditions, salespersonUsername, customerName};
            allSalesRecords.push_back(record);
        }
        file.close();
    }
}

// ฟังก์ชันแสดงประวัติการสั่งซื้อ
void viewOrderHistory() {
    cout << "\n--- Order History ---\n";
    cout << left << setw(12) << "Date" << setw(15) << "model" << setw(15) << "Product" << setw(10) << "Code"
         << setw(10) << "Quantity" << setw(10) << "Price" << setw(15) << "Salesperson" << endl;
    cout << string(87, '-') << endl;

    for (const auto& record : allSalesRecords) {
        cout << left << setw(12) << record.purchaseDate << setw(15) << record.customerName << setw(15) << record.productName
             << setw(10) << record.productCode << setw(10) << record.quantity << setw(10) << fixed << setprecision(2) << record.totalPrice
             << setw(15) << users[record.salespersonUsername].name << endl;
    }
}

// ฟังก์ชันแสดงเมนูย่อยสำหรับ Sales System พร้อมเมนู 'Back'
void showSalesSubmenu(const string& username) {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Sales System Submenu ===\n";
        cout << "1. Record Sale\n2. Generate Receipt\n3. Search Product\n4. View Order History\n0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 4) {
                cout << "Invalid choice. Please enter a number between 0 and 4: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                sellProduct(username);
                break;
            case 2:
                generateReceipt(username); // ส่ง username เข้าไป
                break;
            case 3:
                searchProduct();
                break;
            case 4:
                viewOrderHistory();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันบันทึกการรับประกันสินค้า พร้อมเมนู 'Back'
void recordWarrantyDetails() {
    string code;
    cout << "Enter Product Code (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) == inventory.end()) {
        cout << "Product not found.\n";
        return;
    }

    cin.ignore();  // Clear the input buffer
    cout << "Enter Warranty Period: ";
    getline(cin, inventory[code].warrantyPeriod);
    cout << "Enter Warranty Conditions: ";
    getline(cin, inventory[code].warrantyConditions);

    saveAllProductsToFile();
    cout << "Warranty details recorded successfully!\n";
}

// ฟังก์ชันบันทึกข้อมูลการซ่อมสินค้า พร้อมเมนู 'Back'
void recordRepair() {
    string code, reason, date, status;
    cout << "Enter Product Code (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (inventory.find(code) == inventory.end()) {
        cout << "Product not found.\n";
        return;
    }

    cin.ignore();  // Clear the input buffer
    cout << "Enter Repair Reason: ";
    getline(cin, reason);
    cout << "Enter Repair Date: ";
    getline(cin, date);
    cout << "Enter Repair Status (e.g., 'In Repair', 'Repaired'): ";
    getline(cin, status);

    RepairRecord record = {code, reason, date, status};
    repairHistory[code].push_back(record);

    ofstream file("repair_history.txt", ios::app);
    file << code << "\n" << reason << "\n" << date << "\n" << status << endl;
    file.close();

    cout << "Repair record added successfully!\n";
}

// ฟังก์ชันแก้ไขข้อมูลการซ่อมสินค้า
void editRepairRecord() {
    string code;
    cout << "Enter Product Code to edit repair record (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (repairHistory.find(code) == repairHistory.end()) {
        cout << "No repair records found for this product.\n";
        return;
    }

    vector<RepairRecord>& records = repairHistory[code];

    cout << "\n--- Repair Records for Product Code: " << code << " ---\n";
    for (size_t i = 0; i < records.size(); ++i) {
        cout << i + 1 << ". Date: " << records[i].repairDate << " | Reason: " << records[i].repairReason
             << " | Status: " << records[i].repairStatus << endl;
    }

    int recordNumber;
    cout << "Enter the number of the repair record to edit (or '0' to cancel): ";
    cin >> recordNumber;

    if (recordNumber == 0) {
        return;
    }

    if (recordNumber < 1 || recordNumber > records.size()) {
        cout << "Invalid record number.\n";
        return;
    }

    RepairRecord& recordToEdit = records[recordNumber - 1];

    cout << "Current Repair Status: " << recordToEdit.repairStatus << endl;
    cout << "Enter new Repair Status (or '0' to cancel): ";
    string newStatus;
    cin.ignore(); // Clear input buffer
    getline(cin, newStatus);

    if (newStatus == "0") {
        return;
    }

    recordToEdit.repairStatus = newStatus;

    // บันทึกการเปลี่ยนแปลงลงในไฟล์
    ofstream file("repair_history.txt", ios::trunc);
    for (const auto& pair : repairHistory) {
        const vector<RepairRecord>& recs = pair.second;
        for (const auto& rec : recs) {
            file << rec.code << "\n" << rec.repairReason << "\n" << rec.repairDate << "\n" << rec.repairStatus << endl;
        }
    }
    file.close();

    cout << "Repair record updated successfully!\n";
}

// ฟังก์ชันแสดงประวัติการซ่อมสินค้า พร้อมเมนู 'Back'
void viewRepairHistory() {
    string code;
    cout << "Enter Product Code to view repair history (or '0' to cancel): ";
    cin >> code;

    if (code == "0") {
        return;
    }

    if (repairHistory.find(code) == repairHistory.end()) {
        cout << "No repair history for this product.\n";
        return;
    }

    cout << "\n--- Repair History for Product Code: " << code << " ---\n";
    cout << left << setw(12) << "Date" << setw(20) << "Reason" << setw(15) << "Status" << endl;
    cout << string(47, '-') << endl;
    for (const auto& record : repairHistory[code]) {
        cout << left << setw(12) << record.repairDate << setw(20) << record.repairReason << setw(15) << record.repairStatus << endl;
    }
}

// ฟังก์ชันโหลดประวัติการซ่อมจากไฟล์
void loadRepairHistoryFromFile() {
    ifstream file("repair_history.txt");
    if (file) {
        string code, reason, date, status;
        while (getline(file, code)) {
            getline(file, reason);
            getline(file, date);
            getline(file, status);

            RepairRecord record = {code, reason, date, status};
            repairHistory[code].push_back(record);
        }
        file.close();
    }
}

// ฟังก์ชันแสดงรายงานการซ่อม
void showRepairReport() {
    cout << "\n=== Repair Report ===\n";
    for (const auto& pair : repairHistory) {
        const string& productCode = pair.first;
        const vector<RepairRecord>& records = pair.second;
        cout << "\nProduct Code: " << productCode << endl;
        cout << "Product Name: " << inventory[productCode].name << endl; // แสดงชื่อสินค้า
        cout << left << setw(12) << "Date" << setw(20) << "Reason" << setw(15) << "Status" << endl;
        cout << string(47, '-') << endl;
        for (const auto& record : records) {
            cout << left << setw(12) << record.repairDate << setw(20) << record.repairReason << setw(15) << record.repairStatus << endl;
        }
    }
}

// ฟังก์ชันแสดงเมนูย่อยสำหรับ Warranty and Repair System พร้อมเมนู 'Back'
void showWarrantySubmenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Warranty and Repair System Submenu ===\n";
        cout << "1. Record Warranty Details\n2. Record Repair\n3. View Repair History\n4. Repair Report\n";
        cout << "5. Edit Repair Record\n0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 5) {
                cout << "Invalid choice. Please enter a number between 0 and 5: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                recordWarrantyDetails();
                break;
            case 2:
                recordRepair();
                break;
            case 3:
                viewRepairHistory();
                break;
            case 4:
                showRepairReport();
                break;
            case 5:
                editRepairRecord();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันบันทึกข้อมูลลูกค้า
void recordCustomerInfo(const string& username) {
    Customer customer;
    customer.username = username;
    cout << "Enter full name: ";
    cin.ignore();
    getline(cin, customer.name);
    cout << "Enter address: ";
    getline(cin, customer.address);
    cout << "Enter contact number: ";
    getline(cin, customer.contactNumber);

    customers[username] = customer;

    // บันทึกข้อมูลลูกค้าในไฟล์
    ofstream file("customers.txt", ios::app);
    file << username << "\n" << customer.name << "\n" << customer.address << "\n" << customer.contactNumber << endl;
    file.close();

    cout << "Customer information recorded successfully!\n";
}

// ฟังก์ชันโหลดข้อมูลลูกค้าจากไฟล์
void loadCustomersFromFile() {
    ifstream file("customers.txt");
    if (file) {
        string username, name, address, contactNumber;
        while (getline(file, username)) {
            getline(file, name);
            getline(file, address);
            getline(file, contactNumber);

            Customer customer = {username, name, address, contactNumber};
            customers[username] = customer;
        }
        file.close();
    }
}

// ฟังก์ชันดูข้อมูลลูกค้า
void viewCustomerInfo(const string& username) {
    if (customers.find(username) != customers.end()) {
        Customer& customer = customers[username];
        cout << "\n--- Customer Information ---\n";
        cout << "Username: " << customer.username << "\n";
        cout << "Full Name: " << customer.name << "\n";
        cout << "Address: " << customer.address << "\n";
        cout << "Contact Number: " << customer.contactNumber << "\n";
    } else {
        cout << "No customer information found for this user.\n";
    }
}

// ฟังก์ชันแก้ไขข้อมูลลูกค้า
void editCustomerInfo(const string& username) {
    if (customers.find(username) != customers.end()) {
        Customer& customer = customers[username];
        cout << "\n--- Edit Customer Information ---\n";
        cout << "Current Full Name: " << customer.name << "\n";
        cout << "Enter new Full Name (or press Enter to keep current): ";
        string input;
        cin.ignore();
        getline(cin, input);
        if (!input.empty()) {
            customer.name = input;
        }
        cout << "Current Address: " << customer.address << "\n";
        cout << "Enter new Address (or press Enter to keep current): ";
        getline(cin, input);
        if (!input.empty()) {
            customer.address = input;
        }
        cout << "Current Contact Number: " << customer.contactNumber << "\n";
        cout << "Enter new Contact Number (or press Enter to keep current): ";
        getline(cin, input);
        if (!input.empty()) {
            customer.contactNumber = input;
        }

        // Save updated customer information to file
        ofstream file("customers.txt", ios::trunc);
        for (const auto& pair : customers) {
            const Customer& cust = pair.second;
            file << cust.username << "\n" << cust.name << "\n" << cust.address << "\n" << cust.contactNumber << endl;
        }
        file.close();

        cout << "Customer information updated successfully!\n";
    } else {
        cout << "No customer information found for this user.\n";
    }
}

// ฟังก์ชันโหลดประวัติการซื้อจากไฟล์
void loadPurchaseHistoryFromFile() {
    ifstream file("purchase_history.txt");
    if (file) {
        string username, code, productName, purchaseDate, warrantyPeriod, warrantyConditions;
        int quantity;
        double totalPrice;
        while (getline(file, username)) {
            getline(file, code);
            getline(file, productName);
            file >> quantity;
            file >> totalPrice;
            file.ignore(); // ข้าม newline
            getline(file, purchaseDate);
            getline(file, warrantyPeriod);
            getline(file, warrantyConditions);

            PurchaseRecord record = {username, code, productName, quantity, totalPrice, purchaseDate, warrantyPeriod, warrantyConditions};
            purchaseHistory[username].push_back(record);
        }
        file.close();
    }
}

// ฟังก์ชันดูประวัติการซื้อสินค้า
void viewPurchaseHistory(const string& username) {
    if (purchaseHistory.find(username) != purchaseHistory.end()) {
        cout << "\n--- Purchase History for " << username << " ---\n";
        for (const auto& record : purchaseHistory[username]) {
            cout << "Date: " << record.purchaseDate << " | Product: " << record.productName
                 << " | Code: " << record.productCode
                 << " | Quantity: " << record.quantity << " | Total Price: $" << record.totalPrice << endl;
            cout << "Warranty Period: " << record.warrantyPeriod << " | Warranty Conditions: " << record.warrantyConditions << "\n\n";
        }
    } else {
        cout << "No purchase history for this customer.\n";
    }
}

// ฟังก์ชันให้ลูกค้าซื้อสินค้า พร้อมการตรวจสอบจำนวนสินค้า
void purchaseProducts(const string& username) {
    if (customers.find(username) == customers.end()) {
        cout << "Please record your customer information before making a purchase.\n";
        recordCustomerInfo(username);
    }
    string choice;
    do {
        cout << "\n--- Available Products ---\n";
        for (const auto& item : inventory) {
            const Product& product = item.second;
            cout << "Product Code: " << product.code << " | Name: " << product.name
                 << " | Price: $" << product.price << " | Quantity Available: "
                 << product.quantity << endl;
        }
        cout << "\nEnter the Product Code of the item you wish to purchase (or '0' to return): ";
        cin >> choice;
        if (choice == "0") {
            break;
        }
        if (inventory.find(choice) != inventory.end()) {
            Product& product = inventory[choice];
            int quantity;
            // ตรวจสอบการใส่ค่าให้ถูกต้องสำหรับจำนวนสินค้า
            while (true) {
                cout << "Enter quantity to purchase: ";
                cin >> quantity;
                if (cin.fail() || quantity <= 0) {
                    cout << "Invalid quantity. Please enter a positive number.\n";
                    cin.clear();
                    cin.ignore(numeric_limits<streamsize>::max(), '\n');
                } else if (quantity > product.quantity) {
                    cout << "Insufficient stock. Available: " << product.quantity << endl;
                    cout << "Please enter a new quantity or '0' to cancel.\n";
                    continue;
                } else {
                    break;
                }
            }

            if (quantity == 0) {
                continue;
            }

            product.quantity -= quantity;
            double totalPrice = product.price * quantity;
            saveAllProductsToFile();
            cout << "Purchase successful! Total Price: $" << fixed << setprecision(2) << totalPrice << endl;

            // Record purchase history
            time_t now = time(0);
            tm* ltm = localtime(&now);
            char dateStr[20];
            strftime(dateStr, sizeof(dateStr), "%Y-%m-%d", ltm);
            string purchaseDate(dateStr);

            PurchaseRecord record = {username, product.code, product.name, quantity, totalPrice, purchaseDate, product.warrantyPeriod, product.warrantyConditions};
            purchaseHistory[username].push_back(record);

            // Save purchase history to file
            ofstream file("purchase_history.txt", ios::app);
            file << username << "\n" << product.code << "\n" << product.name << "\n"
                 << quantity << "\n" << totalPrice << "\n" << purchaseDate << "\n"
                 << product.warrantyPeriod << "\n" << product.warrantyConditions << endl;
            file.close();

            // บันทึกประวัติการเข้า-ออกของสินค้า
            recordInventoryChange(product.code, product.name, quantity, "Removed");

            // บันทึกประวัติการขาย
            PurchaseRecord saleRecord = {username, product.code, product.name, quantity, totalPrice, purchaseDate, product.warrantyPeriod, product.warrantyConditions, "Online Purchase", customers[username].name};
            allSalesRecords.push_back(saleRecord);

            // บันทึกลงไฟล์
            ofstream salesFile("all_sales_records.txt", ios::app);
            salesFile << username << "\n" << product.code << "\n" << product.name << "\n"
                      << quantity << "\n" << totalPrice << "\n" << purchaseDate << "\n"
                      << product.warrantyPeriod << "\n" << product.warrantyConditions << "\n"
                      << "Online Purchase" << "\n" << customers[username].name << endl;
            salesFile.close();
        } else {
            cout << "Product not found.\n";
        }
    } while (true);
}

// ฟังก์ชันแสดงเมนูสำหรับลูกค้า พร้อมเมนู 'Back'
void showCustomerMenu(const string& username) {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Customer Menu ===\n";
        cout << "1. View Your Information\n";
        cout << "2. Edit Your Information\n";
        cout << "3. Purchase Products\n";
        cout << "4. View Purchase History\n";
        cout << "0. Logout\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 4) {
                cout << "Invalid choice. Please enter a number between 0 and 4: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                viewCustomerInfo(username);
                break;
            case 2:
                editCustomerInfo(username);
                break;
            case 3:
                purchaseProducts(username);
                break;
            case 4:
                viewPurchaseHistory(username);
                break;
            case 0:
                cout << "Logging out...\n";
                running = false;
                break;
        }
    }
}

// ฟังก์ชันแสดงเมนูสำหรับการจัดการลูกค้า
void showCustomerManagementMenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Customer Management ===\n";
        cout << "1. View All Customers\n";
        cout << "0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 1) {
                cout << "Invalid choice. Please enter 0 or 1: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                viewAllCustomers();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันดูข้อมูลลูกค้าทั้งหมด
void viewAllCustomers() {
    cout << "\n--- All Customers Information ---\n";
    cout << left << setw(15) << "Username" << setw(25) << "Full Name" << setw(30) << "Address" << setw(15) << "Contact" << endl;
    cout << string(85, '-') << endl;
    for (const auto& pair : customers) {
        const Customer& customer = pair.second;
        cout << left << setw(15) << customer.username << setw(25) << customer.name << setw(30) << customer.address << setw(15) << customer.contactNumber << endl;
    }
}

// ฟังก์ชันแสดงรายงานการขาย
void showSalesReport() {
    cout << "\n=== Sales Report ===\n";
    unordered_map<string, int> productSales;
    unordered_map<string, double> productRevenue;
    unordered_map<string, vector<string>> productSaleDates;
    double totalSales = 0.0;

    for (const auto& record : allSalesRecords) {
        productSales[record.productName] += record.quantity;
        productRevenue[record.productName] += record.totalPrice;
        productSaleDates[record.productName].push_back(record.purchaseDate);
        totalSales += record.totalPrice;
    }

    cout << "Total Sales: $" << fixed << setprecision(2) << totalSales << endl;
    cout << "\nProduct Sales:\n";
    for (const auto& pair : productSales) {
        const string& productName = pair.first;
        int quantitySold = pair.second;
        double revenue = productRevenue[productName];
        cout << "Product: " << productName << " | Quantity Sold: " << quantitySold
             << " | Revenue: $" << fixed << setprecision(2) << revenue << endl;
        cout << "Sale Dates: ";
        for (const auto& date : productSaleDates[productName]) {
            cout << date << " ";
        }
        cout << "\n-------------------------\n";
    }

    // หาสินค้าที่ขายดีที่สุด
    auto bestSeller = max_element(productSales.begin(), productSales.end(),
                                  [](const pair<string, int>& a, const pair<string, int>& b) {
                                      return a.second < b.second;
                                  });

    if (bestSeller != productSales.end()) {
        cout << "\nBest-Selling Product: " << bestSeller->first << " | Quantity Sold: " << bestSeller->second << endl;
    }
}

// ฟังก์ชันแสดงรายงานการขายตามช่วงเวลา
void showSalesReportByPeriod() {
    cout << "\nEnter start date (YYYY-MM-DD): ";
    string startDate, endDate;
    cin >> startDate;
    cout << "Enter end date (YYYY-MM-DD): ";
    cin >> endDate;

    unordered_map<string, int> productSales;
    unordered_map<string, double> productRevenue;
    unordered_map<string, vector<string>> productSaleDates;
    double totalSales = 0.0;

    for (const auto& record : allSalesRecords) {
        if (record.purchaseDate >= startDate && record.purchaseDate <= endDate) {
            productSales[record.productName] += record.quantity;
            productRevenue[record.productName] += record.totalPrice;
            productSaleDates[record.productName].push_back(record.purchaseDate);
            totalSales += record.totalPrice;
        }
    }

    cout << "\nSales Report from " << startDate << " to " << endDate << ":\n";
    cout << "Total Sales: $" << fixed << setprecision(2) << totalSales << endl;
    cout << "\nProduct Sales:\n";
    for (const auto& pair : productSales) {
        const string& productName = pair.first;
        int quantitySold = pair.second;
        double revenue = productRevenue[productName];
        cout << "Product: " << productName << " | Quantity Sold: " << quantitySold
             << " | Revenue: $" << fixed << setprecision(2) << revenue << endl;
        cout << "Sale Dates: ";
        for (const auto& date : productSaleDates[productName]) {
            cout << date << " ";
        }
        cout << "\n-------------------------\n";
    }
}

// ฟังก์ชันแสดงรายงานคลังสินค้า
void showInventoryReport() {
    cout << "\n=== Inventory Report ===\n";
    checkStockLevel(); // เรียกใช้ฟังก์ชันที่มีอยู่แล้วในการแสดงสินค้าคงคลัง
    lowStockAlert();    // เรียกใช้ฟังก์ชันแจ้งเตือนสต็อกต่ำ
}

// ฟังก์ชันแสดงเมนูรายงาน พร้อมเมนู 'Back'
void showReportingSystemMenu() {
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Reporting System ===\n";
        cout << "1. Sales Report\n";
        cout << "2. Sales Report by Period\n";
        cout << "3. Inventory Report\n";
        cout << "4. Inventory Change History\n";
        cout << "5. Repair Report\n";
        cout << "0. Back\n";
        cout << "Select an option: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > 5) {
                cout << "Invalid choice. Please enter a number between 0 and 5: ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        switch (choice) {
            case 1:
                showSalesReport();
                break;
            case 2:
                showSalesReportByPeriod();
                break;
            case 3:
                showInventoryReport();
                break;
            case 4:
                showInventoryChangeHistory();
                break;
            case 5:
                showRepairReport();
                break;
            case 0:
                running = false;
                break;
        }
    }
}

// ฟังก์ชันแสดงเมนูหลักและจัดการการเรียกใช้เมนูย่อย พร้อมการตรวจสอบอินพุต
void showUserMenu(const string& username) {
    if (users[username].role == "Customer") {
        showCustomerMenu(username);
        return;
    }
    int choice;
    bool running = true;
    while (running) {
        cout << "\n=== Computer management system mainmenu ===\n";
        int i = 1;

        for (const auto& permission : accessPermissions[username]) {
            cout << i++ << ". " << permission << endl;
        }
        cout << "0. Logout\n";
        cout << "Enter your choice: ";

        // ตรวจสอบการใส่ค่าให้ถูกต้อง
        while (true) {
            cin >> choice;
            if (cin.fail() || choice < 0 || choice > accessPermissions[username].size()) {
                cout << "Invalid choice. Please enter a number between 0 and " << accessPermissions[username].size() << ": ";
                cin.clear();
                cin.ignore(numeric_limits<streamsize>::max(), '\n');
            } else {
                break;
            }
        }

        if (choice == 0) {
            cout << "Logging out...\n";
            running = false;
        } else {
            string selectedPermission = accessPermissions[username][choice - 1];
            if (selectedPermission == "Inventory Management") {
                showInventorySubmenu();
            } else if (selectedPermission == "Sales System") {
                showSalesSubmenu(username);
            } else if (selectedPermission == "Warranty and Repair System") {
                showWarrantySubmenu();
            } else if (selectedPermission == "Customer Management") {
                showCustomerManagementMenu();
            } else if (selectedPermission == "Reporting System") {
                showReportingSystemMenu();
            } else if (selectedPermission == "User Management") {
                showUserManagementMenu();
            } else {
                cout << "Accessing " << selectedPermission << "...\n";
            }
        }
    }
}

// โปรแกรมหลัก
int main() {
    loadUsersFromFile();
    loadInventoryFromFile();
    loadRepairHistoryFromFile();
    loadCustomersFromFile();
    loadPurchaseHistoryFromFile();
    loadAllSalesRecordsFromFile();
    loadInventoryChangesFromFile();
    mainMenu();
    return 0;
}
