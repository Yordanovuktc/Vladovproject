# -- Създаване на базата данни
CREATE SCHEMA IF NOT EXISTS `era_property` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
USE `era_property`;

-- Таблица за брокери
CREATE TABLE  `broker` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NOT NULL,
  `phonenumber` VARCHAR(15) NOT NULL,
  `commission_percentage` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Таблица за клиенти
CREATE TABLE  `customers` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL,
  `number` VARCHAR(15) NOT NULL,
  `price_range` VARCHAR(256) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Таблица за имоти
CREATE TABLE  `properties` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `address` VARCHAR(100) NOT NULL,
  `price` DECIMAL(15,2) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Таблица за транзакции
CREATE TABLE  `transactions` (
  `transaction_id` INT NOT NULL AUTO_INCREMENT,
  `id_customer` INT NOT NULL,
  `id_broker` INT NOT NULL,
  `transaction_date` DATE NOT NULL,
  PRIMARY KEY (`transaction_id`),
  INDEX `id_customer_idx` (`id_customer` ASC),
  INDEX `id_broker_idx` (`id_broker` ASC),
  CONSTRAINT `fk_customer`
    FOREIGN KEY (`id_customer`)
    REFERENCES `customers` (`id`),
  CONSTRAINT `fk_broker`
    FOREIGN KEY (`id_broker`)
    REFERENCES `broker` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Таблица за собственост
CREATE TABLE  `ownership` (
  `customer_id` INT NOT NULL,
  `property_id` INT NOT NULL,
  PRIMARY KEY (`customer_id`, `property_id`),
  INDEX `property_id_idx` (`property_id` ASC),
  CONSTRAINT `fk_customer_ownership`
    FOREIGN KEY (`customer_id`)
    REFERENCES `customers` (`id`),
  CONSTRAINT `fk_property_ownership`
    FOREIGN KEY (`property_id`)
    REFERENCES `properties` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Лог таблица за тригерите
CREATE TABLE  `logs` (
  `log_id` INT NOT NULL AUTO_INCREMENT,
  `action` VARCHAR(50) NOT NULL,
  `customer_id` INT,
  `action_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`log_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

-- Добавяне на начални данни
INSERT INTO `broker` (`name`, `phonenumber`, `commission_percentage`) VALUES 
('Broker 1', '123456789', 2.5),
('Broker 2', '987654321', 3.0);

INSERT INTO `customers` (`name`, `number`, `price_range`) VALUES 
('Customer 1', '111222333', '100000-200000'),
('Customer 2', '444555666', '200000-300000');

INSERT INTO `properties` (`address`, `price`) VALUES 
('123 Main St', 150000.00),
('456 Elm St', 250000.00);

INSERT INTO `transactions` (`id_customer`, `id_broker`, `transaction_date`) VALUES 
(1, 1, '2023-06-01'),
(2, 2, '2023-06-02');

INSERT INTO `ownership` (`customer_id`, `property_id`) VALUES 
(1, 1),
(2, 2);

-- Тригери за таблицата customers
DELIMITER //

-- Before Insert Trigger
CREATE TRIGGER before_customer_insert
BEFORE INSERT ON customers
FOR EACH ROW
BEGIN
    SET NEW.name = UPPER(NEW.name);
END//

-- After Insert Trigger
CREATE TRIGGER after_customer_insert
AFTER INSERT ON customers
FOR EACH ROW
BEGIN
    INSERT INTO logs (action, customer_id, action_time) VALUES ('INSERT', NEW.id, NOW());
END//

-- Before Update Trigger
CREATE TRIGGER before_customer_update
BEFORE UPDATE ON customers
FOR EACH ROW
BEGIN
    SET NEW.name = UPPER(NEW.name);
END//

-- After Update Trigger
CREATE TRIGGER after_customer_update
AFTER UPDATE ON customers
FOR EACH ROW
BEGIN
    INSERT INTO logs (action, customer_id, action_time) VALUES ('UPDATE', NEW.id, NOW());
END//

DELIMITER ;

-- Изглед за всички транзакции с детайли за клиента и брокера
CREATE VIEW transactions_details AS
SELECT 
    t.transaction_id,
    c.name AS customer_name,
    b.name AS broker_name,
    t.transaction_date
FROM 
    transactions t
JOIN 
    customers c ON t.id_customer = c.id
JOIN 
    broker b ON t.id_broker = b.id;

-- Изглед за всички клиенти с техните имоти
CREATE VIEW customer_properties AS
SELECT 
    c.name AS customer_name,
    p.address AS property_address,
    p.price AS property_price
FROM 
    ownership o
JOIN 
    customers c ON o.customer_id = c.id
JOIN 
    properties p ON o.property_id = p.id;

-- Изглед за всички брокери с техните комисионни
CREATE VIEW broker_commissions AS
SELECT 
    b.name AS broker_name,
    SUM(t.transaction_amount * b.commission_percentage / 100) AS total_commission
FROM 
    transactions t
JOIN 
    broker b ON t.id_broker = b.id
GROUP BY 
    b.name;

-- Потребители и роли
CREATE USER 'db_user1'@'localhost' IDENTIFIED BY 'password1';
CREATE USER 'db_user2'@'localhost' IDENTIFIED BY 'password2';

CREATE ROLE 'db_role_database';
CREATE ROLE 'db_role_table';
CREATE ROLE 'db_role_column';

GRANT ALL PRIVILEGES ON `era_property`.* TO 'db_role_database';
GRANT SELECT, INSERT, UPDATE, DELETE ON `era_property`.`customers` TO 'db_role_table';
GRANT SELECT (name, price_range), INSERT (name, number, price_range), UPDATE (price_range) ON `era_property`.`customers` TO 'db_role_column';

GRANT 'db_role_database' TO 'db_user1'@'localhost';
GRANT 'db_role_table' TO 'db_user2'@'localhost';
GRANT 'db_role_column' TO 'db_user2'@'localhost';

-- Индекси
CREATE INDEX idx_broker_phonenumber ON `broker` (`phonenumber`);
CREATE INDEX idx_customer_name_number ON `customers` (`name`, `number`);
CREATE INDEX idx_property_address_prefix ON `properties` (`address`(10));

-- Пример за транзакция
START TRANSACTION;
UPDATE customers SET price_range = '150000-250000' WHERE id = 1;
INSERT INTO transactions (id_customer, id_broker, transaction_date) VALUES (1, 2, CURDATE());
COMMIT;
