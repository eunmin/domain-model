# 도메인 모델 예제

## 수익 인식 예제 (PEAA)

- 트랜잭션 스크립트

```java
public void calculateRevenueRecognitions(long contractNumber)
			throws ApplicationException {
	try {
		ResultSet contracts = db.findContract(contractNumber);
		contracts.next();
		Money totalRevenue = Money.dollars(contracts.getBigDecimal("revenue"));
		MfDate recognitionDate = new MfDate(contracts.getDate("dateSigned"));
		String type = contracts.getString("type");
		if (type.equals("S")) {
			Money[] allocation = totalRevenue.allocate(3);
			db.insertRecognition(contractNumber, allocation[0], recognitionDate);
			db.insertRecognition(contractNumber, allocation[1], recognitionDate.addDays(60));
			db.insertRecognition(contractNumber, allocation[2], recognitionDate.addDays(90));
		} else if (type.equals("W")) {
			db.insertRecognition(contractNumber, totalRevenue, recognitionDate);
		} else if (type.equals("D")) {
			Money[] allocation = totalRevenue.allocate(3);
			db.insertRecognition(contractNumber, allocation[0], recognitionDate);
			db.insertRecognition(contractNumber, allocation[1], recognitionDate.addDays(30));
			db.insertRecognition(contractNumber, allocation[2], recognitionDate.addDays(60));
		}
	} catch (SQLException e) {
		throw new ApplicationException(e);
	}
}
```

- 도메인 모델

```java
Contract contracts = db.findContract(contractNumber);
contracts.calculateRecognitions();
contracts.save();

class Contract {
  public void calculateRecognitions() {
    product.calculateRevenueRecognitions(this);
  }
}

class Product {
  private RecognitionsStrategy recognitionsStrategy;

  public calculateRevenueRecognitions(Contract contract) {
    recognitionsStrategy.calculateRevenueRecognitions(contract);
  }
}

class CompleteRecognitionsStrategy extends RecognitionsStrategy {
  public void calculateRevenueRecognitions(Contract contract) {
    ...
  }
}
```

- 함수형 도메인 모델 (고민중 - 일단 절차적으로 작성)

```clojure
(defn add-days [day days]
  (+ day days))

(defn three-way-recognition [first-offset second-offset contracts]
  (map (fn [offset]
         {:contract-number (:contract-number contracts)
          :revenue (/ (:revenue contracts) 3)
          :recognition-date (add-days (:date-signed contracts) offset)})
       [0 first-offset second-offset]))

(defn complete-recognition [contracts]
  {:contract-number (:contract-number contracts)
   :revenue (:revenue contracts)
   :recognition-date (:date-signed contracts)})

(def recognition-strategies {"S" (partial three-way-recognition 60 90)
                             "W" complete-recognition
                             "D" (partial three-way-recognition 30 60)})

(defn find-contract [contract-number]
  {:revenue 200
   :date-signed 1
   :type "S"})

(defn insert-recognition [recognition]
  (println recognition)
  recognition)

(defn calculate-revenue-recognitions' [contracts]
  ((get recognition-strategies (:type contracts)) contracts))

(defn calculate-revenue-recognitions [contract-number]
  (->> (find-contract contract-number)
       calculate-revenue-recognitions'
       (map insert-recognition)))

(calculate-revenue-recognitions 1)
```
