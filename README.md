# pl-sql_mini-project
CREATE TABLE customers (
    customer_id NUMBER PRIMARY KEY,
    customer_name VARCHAR2(100) NOT NULL,
    phone VARCHAR2(15) UNIQUE,
    created_at DATE DEFAULT SYSDATE
);

CREATE TABLE bank_accounts (
    account_id NUMBER PRIMARY KEY,
    customer_id NUMBER NOT NULL,
    account_type VARCHAR2(20) DEFAULT 'SAVINGS',
    balance NUMBER(12,2) DEFAULT 0 NOT NULL,
    account_status VARCHAR2(10) DEFAULT 'ACTIVE',
    CONSTRAINT fk_account_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id),
    CONSTRAINT chk_balance CHECK (balance >= 0),
    CONSTRAINT chk_account_status
        CHECK (account_status IN ('ACTIVE', 'BLOCKED'))
);

CREATE TABLE bank_transactions (
    transaction_id NUMBER PRIMARY KEY,
    account_id NUMBER NOT NULL,
    transaction_type VARCHAR2(15) NOT NULL,
    amount NUMBER(12,2) NOT NULL,
    risk_score NUMBER(3) DEFAULT 0,
    transaction_status VARCHAR2(15) NOT NULL,
    transaction_date DATE DEFAULT SYSDATE,
    CONSTRAINT fk_txn_account
        FOREIGN KEY (account_id)
        REFERENCES bank_accounts(account_id),
    CONSTRAINT chk_txn_amount CHECK (amount > 0),
    CONSTRAINT chk_txn_type
        CHECK (transaction_type IN ('DEPOSIT', 'WITHDRAWAL')),
    CONSTRAINT chk_txn_status
        CHECK (transaction_status IN ('COMPLETED', 'REVIEW'))
);

CREATE TABLE fraud_rules (
    rule_code VARCHAR2(30) PRIMARY KEY,
    threshold_amount NUMBER(12,2) NOT NULL,
    score_points NUMBER(3) NOT NULL,
    active CHAR(1) DEFAULT 'Y',
    CONSTRAINT chk_rule_active CHECK (active IN ('Y', 'N')),
    CONSTRAINT chk_rule_score CHECK (score_points BETWEEN 0 AND 100)
);

CREATE TABLE fraud_alerts (
    alert_id NUMBER PRIMARY KEY,
    transaction_id NUMBER NOT NULL,
    risk_score NUMBER(3) NOT NULL,
    alert_message VARCHAR2(250),
    alert_status VARCHAR2(15) DEFAULT 'OPEN',
    created_at DATE DEFAULT SYSDATE,
    CONSTRAINT fk_alert_transaction
        FOREIGN KEY (transaction_id)
        REFERENCES bank_transactions(transaction_id),
    CONSTRAINT chk_alert_status
        CHECK (alert_status IN ('OPEN', 'REVIEWED', 'CLOSED'))
);

CREATE TABLE audit_logs (
    log_id NUMBER PRIMARY KEY,
    transaction_id NUMBER,
    action_description VARCHAR2(250),
    action_date DATE DEFAULT SYSDATE
);
CREATE SEQUENCE txn_seq START WITH 1001 INCREMENT BY 1;
CREATE SEQUENCE alert_seq START WITH 501 INCREMENT BY 1;
CREATE SEQUENCE audit_seq START WITH 1 INCREMENT BY 1;
INSERT INTO customers
VALUES (1, 'Aman Sharma', '9876500001', SYSDATE);

INSERT INTO customers
VALUES (2, 'Priya Verma', '9876500002', SYSDATE);

INSERT INTO customers
VALUES (3, 'Rahul Singh', '9876500003', SYSDATE);

INSERT INTO bank_accounts
VALUES (101, 1, 'SAVINGS', 100000, 'ACTIVE');

INSERT INTO bank_accounts
VALUES (102, 2, 'SAVINGS', 25000, 'ACTIVE');

INSERT INTO bank_accounts
VALUES (103, 3, 'SAVINGS', 50000, 'ACTIVE');

INSERT INTO fraud_rules VALUES ('AMOUNT_25K', 25000, 25, 'Y');
INSERT INTO fraud_rules VALUES ('AMOUNT_50K', 50000, 50, 'Y');
INSERT INTO fraud_rules VALUES ('AMOUNT_75K', 75000, 70, 'Y');
INSERT INTO fraud_rules VALUES ('RAPID_10MIN', 2, 30, 'Y');

COMMIT;
SELECT * FROM customers;
SELECT * FROM bank_accounts;
SELECT * FROM fraud_rules;
CREATE OR REPLACE FUNCTION calculate_risk_score (
    p_account_id IN NUMBER,
    p_amount     IN NUMBER
)
RETURN NUMBER
IS
    v_amount_score NUMBER := 0;
    v_rapid_score  NUMBER := 0;
    v_recent_count NUMBER := 0;
BEGIN
    SELECT NVL(MAX(score_points), 0)
    INTO v_amount_score
    FROM fraud_rules
    WHERE rule_code LIKE 'AMOUNT%'
      AND active = 'Y'
      AND threshold_amount <= p_amount;

    SELECT COUNT(*)
    INTO v_recent_count
    FROM bank_transactions
    WHERE account_id = p_account_id
      AND transaction_date >= SYSDATE - (10 / 1440);

    SELECT NVL(MAX(score_points), 0)
    INTO v_rapid_score
    FROM fraud_rules
    WHERE rule_code = 'RAPID_10MIN'
      AND active = 'Y'
      AND v_recent_count >= threshold_amount;

    RETURN LEAST(100, v_amount_score + v_rapid_score);
END;
/
SELECT calculate_risk_score(101, 10000) AS risk_score
FROM dual;

SELECT calculate_risk_score(101, 80000) AS risk_score
FROM dual;
CREATE OR REPLACE TRIGGER trg_transaction_audit
AFTER INSERT ON bank_transactions
FOR EACH ROW
BEGIN
    INSERT INTO audit_logs (
        log_id,
        transaction_id,
        action_description,
        action_date
    )
    VALUES (
        audit_seq.NEXTVAL,
        :NEW.transaction_id,
        'Transaction created with status: '
            || :NEW.transaction_status,
        SYSDATE
    );
END;
/
SELECT * FROM audit_logs;
CREATE OR REPLACE PACKAGE sentinel_pkg AS

    PROCEDURE process_transaction (
        p_account_id     IN NUMBER,
        p_type           IN VARCHAR2,
        p_amount         IN NUMBER,
        p_transaction_id OUT NUMBER
    );

END sentinel_pkg;
/
CREATE OR REPLACE PACKAGE BODY sentinel_pkg AS

    PROCEDURE process_transaction (
        p_account_id     IN NUMBER,
        p_type           IN VARCHAR2,
        p_amount         IN NUMBER,
        p_transaction_id OUT NUMBER
    )
    IS
        v_balance        NUMBER(12,2);
        v_account_status VARCHAR2(10);
        v_risk_score     NUMBER;
        v_txn_status     VARCHAR2(15);
        v_type           VARCHAR2(15);
    BEGIN
        v_type := UPPER(TRIM(p_type));

        IF p_amount IS NULL OR p_amount <= 0 THEN
            RAISE_APPLICATION_ERROR(
                -20001, 'Amount must be greater than zero.'
            );
        END IF;

        IF v_type NOT IN ('DEPOSIT', 'WITHDRAWAL') THEN
            RAISE_APPLICATION_ERROR(
                -20002, 'Invalid transaction type.'
            );
        END IF;

        SELECT balance, account_status
        INTO v_balance, v_account_status
        FROM bank_accounts
        WHERE account_id = p_account_id
        FOR UPDATE;

        IF v_account_status <> 'ACTIVE' THEN
            RAISE_APPLICATION_ERROR(
                -20004, 'Account is not active.'
            );
        END IF;

        IF v_type = 'WITHDRAWAL'
           AND v_balance < p_amount THEN
            RAISE_APPLICATION_ERROR(
                -20005, 'Insufficient account balance.'
            );
        END IF;

        v_risk_score :=
            calculate_risk_score(p_account_id, p_amount);

        IF v_risk_score >= 70 THEN
            v_txn_status := 'REVIEW';
        ELSE
            v_txn_status := 'COMPLETED';
        END IF;

        -- Only completed transactions change the balance.
        IF v_txn_status = 'COMPLETED' THEN

            IF v_type = 'DEPOSIT' THEN
                UPDATE bank_accounts
                SET balance = balance + p_amount
                WHERE account_id = p_account_id;
            ELSE
                UPDATE bank_accounts
                SET balance = balance - p_amount
                WHERE account_id = p_account_id;
            END IF;

        END IF;

        p_transaction_id := txn_seq.NEXTVAL;

        INSERT INTO bank_transactions (
            transaction_id,
            account_id,
            transaction_type,
            amount,
            risk_score,
            transaction_status,
            transaction_date
        )
        VALUES (
            p_transaction_id,
            p_account_id,
            v_type,
            p_amount,
            v_risk_score,
            v_txn_status,
            SYSDATE
        );

        IF v_txn_status = 'REVIEW' THEN

            INSERT INTO fraud_alerts (
                alert_id,
                transaction_id,
                risk_score,
                alert_message,
                alert_status,
                created_at
            )
            VALUES (
                alert_seq.NEXTVAL,
                p_transaction_id,
                v_risk_score,
                'High-risk transaction requires manual review.',
                'OPEN',
                SYSDATE
            );

        END IF;

        COMMIT;

    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            ROLLBACK;
            RAISE_APPLICATION_ERROR(
                -20003, 'Account not found.'
            );

        WHEN OTHERS THEN
            ROLLBACK;
            RAISE;

    END process_transaction;

END sentinel_pkg;
/
DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id     => 101,
        p_type           => 'DEPOSIT',
        p_amount         => 5000,
        p_transaction_id => v_txn_id
    );
END;
/
DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id     => 101,
        p_type           => 'DEPOSIT',
        p_amount         => 80000,
        p_transaction_id => v_txn_id
    );
END;
/
DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id     => 102,
        p_type           => 'WITHDRAWAL',
        p_amount         => 30000,
        p_transaction_id => v_txn_id
    );
END;
/
DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id     => 101,
        p_type           => 'PAYMENT',
        p_amount         => 1000,
        p_transaction_id => v_txn_id
    );
END;
/
SELECT
    transaction_id,
    account_id,
    transaction_type,
    amount,
    risk_score,
    transaction_status,
    transaction_date
FROM bank_transactions
ORDER BY transaction_id;
SELECT
    a.alert_id,
    t.account_id,
    t.amount,
    a.risk_score,
    a.alert_message,
    a.alert_status
FROM fraud_alerts a
JOIN bank_transactions t
    ON a.transaction_id = t.transaction_id;
    SELECT
    account_id,
    balance,
    account_status
FROM bank_accounts
ORDER BY account_id;
SELECT
    log_id,
    transaction_id,
    action_description,
    action_date
FROM audit_logs
ORDER BY log_id;
SELECT
    CASE
        WHEN risk_score >= 70 THEN 'HIGH'
        WHEN risk_score >= 30 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS risk_category,
    COUNT(*) AS total_transactions
FROM bank_transactions
GROUP BY
    CASE
        WHEN risk_score >= 70 THEN 'HIGH'
        WHEN risk_score >= 30 THEN 'MEDIUM'
        ELSE 'LOW'
    END;
    SELECT name, type, line, position, text
FROM user_errors
WHERE name IN (
    'CALCULATE_RISK_SCORE',
    'TRG_TRANSACTION_AUDIT',
    'SENTINEL_PKG'
)
ORDER BY name, sequence;
SELECT object_name, object_type, status
FROM user_objects
WHERE object_name IN (
    'CUSTOMERS',
    'BANK_ACCOUNTS',
    'BANK_TRANSACTIONS',
    'FRAUD_RULES',
    'FRAUD_ALERTS',
    'AUDIT_LOGS',
    'CALCULATE_RISK_SCORE',
    'TRG_TRANSACTION_AUDIT',
    'SENTINEL_PKG'
)
ORDER BY object_type, object_name;
SELECT * FROM bank_accounts;
SELECT * FROM fraud_rules;
DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id => 101,
        p_type => 'DEPOSIT',
        p_amount => 5000,
        p_transaction_id => v_txn_id
    );

    DBMS_OUTPUT.PUT_LINE('Transaction ID: ' || v_txn_id);
    DBMS_OUTPUT.PUT_LINE('Status: COMPLETED');
END;
/
SELECT transaction_id, account_id, transaction_type,
       amount, risk_score, transaction_status
FROM bank_transactions
ORDER BY transaction_id;

DECLARE
    v_txn_id NUMBER;
BEGIN
    sentinel_pkg.process_transaction(
        p_account_id => 101,
        p_type => 'DEPOSIT',
        p_amount => 80000,
        p_transaction_id => v_txn_id
    );

    DBMS_OUTPUT.PUT_LINE('Transaction ID: ' || v_txn_id);
    DBMS_OUTPUT.PUT_LINE('Status: REVIEW');
    DBMS_OUTPUT.PUT_LINE('High Risk Transaction Detected!');
END;
/

SELECT
    a.alert_id,
    a.transaction_id,
    t.account_id,
    t.amount,
    a.risk_score,
    a.alert_message,
    a.alert_status
FROM fraud_alerts a
JOIN bank_transactions t
    ON a.transaction_id = t.transaction_id;

SELECT
    log_id,
    transaction_id,
    action_description,
    action_date
FROM audit_logs
ORDER BY log_id;

SELECT
    c.customer_name,
    a.account_id,
    t.transaction_id,
    t.transaction_type,
    t.amount,
    t.risk_score,
    t.transaction_status,
    CASE
        WHEN t.risk_score >= 70 THEN 'HIGH RISK'
        WHEN t.risk_score >= 25 THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END AS risk_level
FROM bank_transactions t
JOIN bank_accounts a
    ON t.account_id = a.account_id
JOIN customers c
    ON a.customer_id = c.customer_id
ORDER BY t.transaction_id;

SELECT
    transaction_id,
    account_id,
    transaction_type,
    amount,
    risk_score,
    transaction_status
FROM bank_transactions
WHERE risk_score < 70
  AND transaction_status = 'COMPLETED'
ORDER BY transaction_id;
SELECT
    alert_id,
    transaction_id,
    risk_score,
    alert_status,
    alert_message,
    created_at
FROM fraud_alerts
ORDER BY alert_id;
SELECT
    audit_id,
    transaction_id,
    action_type,
    action_details,
    created_at
FROM audit_logs
ORDER BY audit_id;
SELECT *
FROM audit_logs;
