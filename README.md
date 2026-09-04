สรุปทั้ง 9 ตาราง
** ตารางที่ 1 owner (เจ้าของสัตว์เลี้ยง) **
** ตารางที่ 2 owner_phone (เบอร์โทรศัพท์เจ้าของสัตว์เลี้ยง) **
** ตารางที่ 3 animal (ข้อมูลสัว์เลี้ยง) **
** ตารางที่ 4 vet (ข้อมูลของสัตวแพทย์) **
** ตารางที่ 5 visit (ประวัติการรักษา) **
** ตารางที่ 6 vet_specialty (ความเชี่ยวชาญของสัตวแพทย์)**
** ตารางที่ 7 medicine (ยา)**
** ตารางที่ 8 visit_medicine (เพิ่มประวัติการจ่ายยา) **
** ตารางที่ 9 receipt (ใบเสร็จ) **


DROP TABLE IF EXISTS receipt CASCADE;
DROP TABLE IF EXISTS visit_medicine CASCADE;
DROP TABLE IF EXISTS medicine CASCADE;
DROP TABLE IF EXISTS vet_specialty CASCADE;
DROP TABLE IF EXISTS visit CASCADE;
DROP TABLE IF EXISTS vet CASCADE;
DROP TABLE IF EXISTS animal CASCADE;
DROP TABLE IF EXISTS owner_phone CASCADE;
DROP TABLE IF EXISTS owner CASCADE;


-- 1. ตาราง owner (เจ้าของสัตว์)
-- มาจากเอนทิติ: owner

CREATE TABLE owner (
	owner_id SERIAL,
	first_name VARCHAR(100)		NOT NULL,
	last_name VARCHAR(100)		NOT NULL,
	address TEXT,
	CONSTRAINT pk_owner			PRIMARY KEY (owner_id)
);


-- 2. ตาราง owner_phone (เบอร์โทรศัพท์เจ้าของสัตว์)
-- มาจากแอตทริบิวต์ (Multivalued Attribute) ของ owner 
CREATE TABLE owner_phone (
    owner_id INT NOT NULL,
    phone_no VARCHAR(20) NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE,
    CONSTRAINT pk_owner_phone PRIMARY KEY (owner_id, phone_no),
    -- เหตุผล ON DELETE CASCADE: เมื่อลบข้อมูลเจ้าของ เบอร์โทรศัพท์ที่ผูกกับเจ้าของคนนี้จะไม่มีความหมายอีกต่อไป จึงต้องลบตามทันที
    CONSTRAINT fk_owner_phone_owner FOREIGN KEY (owner_id) 
        REFERENCES owner(owner_id) ON DELETE CASCADE
);


-- 3. ตาราง animal (สัตว์เลี้ยง)
-- มาจากเอนทิตี: animal (1:M กับ owner โดยวาง FK ถึง M คือ animal)

CREATE TABLE animal(
	animal_id SERIAL,
	owner_id INT		NOT NULL,
	name VARCHAR(100)	NOT NULL,
	species VARCHAR(50)	NOT NULL,
	sex VARCHAR(10),
	color VARCHAR(50),
	birth_date DATE,
	CONSTRAINT pk_animal 			PRIMARY KEY (animal_id),
	-- เหตุผล ON DELETE RESTEICT: ห้ามลบเจ้าของสัตว์ หากยังมีสัตว์เลี้ยงผูกไว้ในระบบ
	CONSTRAINT fk_animal_owner		FOREIGN KEY (owner_id)
		REFERENCES owner(owner_id) 	ON DELETE RESTRICT,
	CONSTRAINT ck_animal_sex CHECK (sex IN ('male', 'Female', 'Unknown'))
);


-- 4. ตาราง vet (สัตวแพทย์)
-- มาจากเอนทิตี: vet

CREATE TABLE vet (
	vet_id SERIAL,
	license_no VARCHAR(50)			NOT NULL,
	name VARCHAR(100)				NOT NULL,
	strart_date DATE,
	CONSTRAINT pk_vet				PRIMARY KEY (vet_id),
	CONSTRAINT uq_vet_license_no 	UNIQUE (license_no)
);


-- 5. ตาราง visit (การเข้ารับบริการ)
-- มาจากเอนทิตีอ่อน (Weak Entity) ของ animal -> ใช้ PK ผสม

CREATE TABLE visit (
	animal_id INT			NOT NULL,
	visit_no INT			NOT NULL,
	visit_date TIMESTAMP 	NOT NULL DEFAULT CURRENT_TIMESTAMP,
	weight NUMERIC(5,2),
	temperature NUMERIC(4,1),
	symptom TEXT,
	diagnosis TEXT,
	vet_id INT,
	CONSTRAINT pk_visit PRIMARY KEY (animal_id, visit_no),
	-- เหตุผล NO DELETE CAScADE: เนื่องจาก visit เป็นเอนทิตีอ่อน ขึ้นตรงกับ animal
	CONSTRAINT fk_visit_animal FOREIGN KEY (animal_id)
		REFERENCES animal(animal_id) ON DELETE CASCADE,
	-- เหตุผล NO DELETE SET NULL: หากสัตวแพทย์ย้ายออกหรือถูกลบออกจากระบบ ประวัติรักษายังคงอยู่
	CONSTRAINT fk_visit_vet FOREIGN KEY (vet_id)
		REFERENCES vet(vet_id) ON DELETE SET NULL,
	CONSTRAINT ck_visit_weight CHECK (weight > 0),
	CONSTRAINT ck_visit_temperature CHECK (temperature BETWEEN 30.0 AND 45.0)
);


-- 6. สร้างตาราง vet_specialty (ความเชี่ยวชาญของสัตวแพทย์)
-- มาจาความสัมพันธ์แบบ M:N ระหว่าง vet กับ specialty

CREATE TABLE vet_specialty (
	vet_id INT NOT NULL,
	specialty VARCHAR(100) NOT NULL,
	CONSTRAINT pk_vet_specialty PRIMARY KEY (vet_id, specialty),
	-- เหตุผล ON DELETE CASCADE: เมื่อลบข้อมูลสัตวแพทย์ รายการความเชี่ยวชาญของสัตวแพทย์คนนั้นจะถูกลบออก
	CONSTRAINT fk_vet_specailty_vet FOREIGN KEY (vet_id)
		REFERENCES vet(vet_id) ON DELETE CASCADE
);


-- 7. สร้างตาราง visit_medicine (ยา)
-- มาจากเอนทิตี: medicine

CREATE TABLE medicine (
	medicine_id SERIAL,
	name VARCHAR(100) NOT NULL,
	unit VARCHAR(20) NOT NULL,
	unit_price NUMERIC(10,2) NOT NULL,
	CONSTRAINT pk_medicine PRIMARY KEY (medicine_id),
	CONSTRAINT ck_medicine_unit_price CHECK (unit_price >= 0)
);


-- 8. สร้างตาราง visit_medicine (การจ่ายยา)
-- มาจากความสัมพันธ์แบบ M:N ระหว่าง visit กับ medicine (ตารางเชื่อม)

CCREATE TABLE visit_medicine (
    animal_id INT,
    visit_no INT,
    medicine_id INT REFERENCES medicine(medicine_id) ON DELETE RESTRICT,
    dosage VARCHAR(100),
    days INT DEFAULT 1,
    PRIMARY KEY (animal_id, visit_no, medicine_id),
    FOREIGN KEY (animal_id, visit_no) REFERENCES visit(animal_id, visit_no) ON DELETE CASCADE
);


-- 9. สร้างตาราง receipt (ใบเสร็จ)
-- มาจากความสัมพันธ์แบบ 1:1 กับ visit (บังคับ Unique Constraint บน FK ผสม)

CREATE TABLE receipt (
    receipt_id    SERIAL,
    animal_id     INT                      NOT NULL,
    visit_no      INT                      NOT NULL,
    paid_at       TIMESTAMP                NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount  NUMERIC(10,2)            NOT NULL,
    CONSTRAINT pk_receipt PRIMARY KEY (receipt_id),
    -- ความสัมพันธ์ 1:1 บังคับให้ใช้ได้แค่ 1 visit ออกได้เพียง 1 ใบเสร็จ เท่านั้น
    CONSTRAINT uq_receipt_visit UNIQUE (animal_id, visit_no),
    -- เหตุผล ON DELETE RESTRICT: ห้ามลบรายการ visit หากมีการออกใบเสร็จและชำระเงินแล้ว
    CONSTRAINT fk_receipt_visit FOREIGN KEY (animal_id, visit_no)
        REFERENCES visit(animal_id, visit_no) ON DELETE RESTRICT,
    CONSTRAINT ck_receipt_total_amount CHECK (total_amount >= 0)
);


INSERT INTO owner (first_name, last_name, address) VALUES
('สมชาย', 'ใจดี', '123/45 กทม.'),
('วิภา', 'รักสัตว์', '99/1 เชียงใหม่');

INSERT INTO owner_phone (owner_id, phone_no, is_primary) VALUES
(1, '081-111-2222', TRUE),
(2, '089-999-8888', TRUE);

INSERT INTO animal (owner_id, name, species, sex, color, birth_date) VALUES
(1, 'ด่าง', 'Dog', 'Male', 'White-Black', '2021-05-10'),
(2, 'เหมียว', 'Cat', 'Female', 'Orange', '2022-01-15');

INSERT INTO vet (license_no, name, start_date) VALUES
('VET-001', 'น.สพ. สมศักดิ์ เก่งกาจ', '2020-01-01'),
('VET-002', 'สพ.ญ. อารี มีสุข', '2021-06-15');

INSERT INTO visit (animal_id, visit_no, weight, temperature, symptom, diagnosis, vet_id) VALUES
(1, 1, 12.50, 38.5, 'ซึม ไม่กินอาหาร', 'พยาธิเม็ดเลือด', 1),
(2, 1, 4.20, 39.0, 'ฉีดวัคซีนประจำปี', 'สุขภาพแข็งแรงดี', 2);

INSERT INTO medicine (name, unit, unit_price) VALUES
('Doxycycline', 'เม็ด', 10.00),
('Rabies Vaccine', 'เข็ม', 150.00);

INSERT INTO visit_medicine (animal_id, visit_no, medicine_id, dosage, days) VALUES
(1, 1, 1, '1 เม็ด หลังอาหาร เช้า-เย็น', 7),
(2, 1, 2, '1 เข็ม', 1);

INSERT INTO receipt (animal_id, visit_no, total_amount) VALUES
(1, 1, 570.00),
(2, 1, 350.00);

SELECT * FROM visit;

<img width="916" height="190" alt="image" src="https://github.com/user-attachments/assets/1e8de72c-3b76-459c-b9d2-9b127ff2a66d" />
