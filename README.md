# furnitureshop
DROP DATABASE IF EXISTS FurnitureStore;
CREATE DATABASE FurnitureStore CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE FurnitureStore;

-- ==========================================
-- БЛОК 1: СПРАВОЧНИКИ
-- ==========================================

CREATE TABLE Roles (
    RoleID INT AUTO_INCREMENT PRIMARY KEY,
    RoleName VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE Categories (
    CategoryID INT AUTO_INCREMENT PRIMARY KEY,
    CategoryName VARCHAR(100) NOT NULL UNIQUE
);

-- НОВАЯ ТАБЛИЦА: Скидки (Акции)
CREATE TABLE Discounts (
    DiscountID INT AUTO_INCREMENT PRIMARY KEY,
    Name VARCHAR(100) NOT NULL,      -- Например: "Осенняя распродажа"
    Percent INT NOT NULL,            -- Размер скидки (например, 15)
    StartDate DATE NOT NULL,         -- Когда начинается
    EndDate DATE NOT NULL,           -- Когда заканчивается
    IsActive BOOLEAN DEFAULT TRUE,   -- Можно отключить вручную
    CHECK (Percent BETWEEN 1 AND 90) -- Защита от скидки 100% или больше
);

-- ==========================================
-- БЛОК 2: ОСНОВНЫЕ СУЩНОСТИ
-- ==========================================

CREATE TABLE Users (
    UserID INT AUTO_INCREMENT PRIMARY KEY,
    Login VARCHAR(50) NOT NULL UNIQUE,
    Password VARCHAR(255) NOT NULL,
    FullName VARCHAR(100) NOT NULL,
    RoleID INT,
    FOREIGN KEY (RoleID) REFERENCES Roles(RoleID) ON DELETE SET NULL
);

CREATE TABLE Customers (
    CustomerID INT AUTO_INCREMENT PRIMARY KEY,
    FullName VARCHAR(150) NOT NULL,
    Phone VARCHAR(20),
    Email VARCHAR(100),
    Address VARCHAR(255)
);

-- ТОВАРЫ (Добавлена ссылка на скидку)
CREATE TABLE Products (
    ProductID INT AUTO_INCREMENT PRIMARY KEY,
    Name VARCHAR(150) NOT NULL,
    Price DECIMAL(10, 2) NOT NULL,      -- Базовая цена (без скидки)
    CategoryID INT,
    DiscountID INT DEFAULT NULL,        -- Ссылка на акцию (если есть)
    Description TEXT,
    FOREIGN KEY (CategoryID) REFERENCES Categories(CategoryID) ON DELETE SET NULL,
    FOREIGN KEY (DiscountID) REFERENCES Discounts(DiscountID) ON DELETE SET NULL
);

CREATE TABLE Stock (
    StockID INT AUTO_INCREMENT PRIMARY KEY,
    ProductID INT NOT NULL UNIQUE,
    Quantity INT DEFAULT 0,
    LastUpdate TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (ProductID) REFERENCES Products(ProductID) ON DELETE CASCADE
);

-- ==========================================
-- БЛОК 3: ТРАНЗАКЦИИ
-- ==========================================

CREATE TABLE Orders (
    OrderID INT AUTO_INCREMENT PRIMARY KEY,
    OrderDate DATETIME DEFAULT CURRENT_TIMESTAMP,
    CustomerID INT NOT NULL,
    UserID INT NOT NULL,
    Status VARCHAR(50) DEFAULT 'Новый',
    TotalAmount DECIMAL(10, 2) DEFAULT 0,
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID),
    FOREIGN KEY (UserID) REFERENCES Users(UserID)
);

CREATE TABLE OrderItems (
    OrderItemID INT AUTO_INCREMENT PRIMARY KEY,
    OrderID INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity INT NOT NULL DEFAULT 1,
    PriceAtSale DECIMAL(10, 2) NOT NULL, -- Здесь сохраняем цену УЖЕ со скидкой
    FOREIGN KEY (OrderID) REFERENCES Orders(OrderID) ON DELETE CASCADE,
    FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
);

-- ==========================================
-- БЛОК 4: ЗАПОЛНЕНИЕ ДАННЫМИ
-- ==========================================

-- 1. Роли
INSERT INTO Roles (RoleName) VALUES 
('Администратор'), ('Менеджер'), ('Кладовщик');

-- 2. Категории
INSERT INTO Categories (CategoryName) VALUES 
('Корпусная мебель'), ('Мягкая мебель'), ('Офисная мебель'), ('Фурнитура');

-- 3. Скидки (Создаем акции)
INSERT INTO Discounts (Name, Percent, StartDate, EndDate, IsActive) VALUES 
('Летняя распродажа', 15, '2024-06-01', '2024-08-31', TRUE),
('Черная пятница', 30, '2024-11-20', '2024-11-27', FALSE), -- Выключена
('Уценка склада', 50, '2023-01-01', '2025-12-31', TRUE); -- Долгая акция

-- 4. Пользователи
INSERT INTO Users (Login, Password, FullName, RoleID) VALUES 
('admin', '123', 'Иванов И.И.', 1),
('manager1', 'qwerty', 'Петрова А.С.', 2);

-- 5. Покупатели
INSERT INTO Customers (FullName, Phone, Email) VALUES 
('Смирнов Алексей', '+7(999)111-22-33', 'smirnov@mail.ru'),
('ООО "Ромашка"', '+7(495)000-11-22', 'info@romashka.ru');

-- 6. Товары (Привязываем к скидкам)
-- Товар 1: Шкаф. Цена 25000. Есть скидка "Летняя распродажа" (15%). Итог: 21250.
-- Товар 2: Диван. Цена 35000. Скидки нет (NULL). Итог: 35000.
-- Товар 3: Стол. Цена 8000. Скидка "Уценка склада" (50%). Итог: 4000.
-- Товар 4: Кресло. Цена 5500. Скидка "Черная пятница" (30%), но она ВЫКЛЮЧЕНА (IsActive = FALSE). Итог: 5500.

INSERT INTO Products (Name, Price, CategoryID, DiscountID, Description) VALUES 
('Шкаф-купе "Модерн"', 25000.00, 1, 1, 'Дуб сонома'),
('Диван-кровать "Комфорт"', 35000.00, 2, NULL, 'Ткань рогожка'),
('Стол письменный', 8000.00, 3, 3, 'ЛДСП, белый'),
('Кресло офисное', 5500.00, 3, 2, 'Кожзам, черный');

-- 7. Склад
INSERT INTO Stock (ProductID, Quantity) VALUES 
(1, 5), (2, 3), (3, 20), (4, 10);

-- 8. Заказы
-- Заказ 1 (Смирнов, менеджер Петрова). Купил Шкаф (со скидкой) и Стол (со скидкой).
INSERT INTO Orders (OrderDate, CustomerID, UserID, Status, TotalAmount) VALUES 
('2024-06-15 14:30:00', 1, 2, 'Выполнен', 25250.00); 
-- Расчет: Шкаф 21250 + Стол 4000 = 25250

INSERT INTO OrderItems (OrderID, ProductID, Quantity, PriceAtSale) VALUES 
(1, 1, 1, 21250.00), -- Шкаф: 25000 - 15% = 21250
(1, 3, 1, 4000.00);  -- Стол: 8000 - 50% = 4000

-- Заказ 2 (Ромашка, менеджер Петрова). Купили 2 дивана (без скидки).
INSERT INTO Orders (OrderDate, CustomerID, UserID, Status, TotalAmount) VALUES 
('2024-06-16 10:00:00', 2, 2, 'В обработке', 70000.00);

INSERT INTO OrderItems (OrderID, ProductID, Quantity, PriceAtSale) VALUES 
(2, 2, 2, 35000.00);
