# Карта кода: как считается решение approve / review / reject

## Участники (backend/src/Domain + backend/config/rules.php)

- `AssessmentService` — оркестратор: валидация → LTV → решение → лимит.
- `ApplicationValidator` (+ `VinValidator`, `VehicleAge`) — нормализация и проверка входа.
- `LtvCalculator` — расчёт LTV.
- `DecisionEngine` — решение по порогам LTV.
- `backend/config/rules.php` — пороги: `ltv.approve_max` = 60.0, `ltv.review_max` = 85.0; остальные справочники (vin, vehicle, amount, term).

## Порядок вызовов в `AssessmentService::assess()` (строки 28–42)

1. `ApplicationValidator::validate($payload)` — нормализует и проверяет вход:
   - VIN через `VinValidator::isValid` (длина 17, `A-Z0-9`, без `I`, `O`, `Q`);
   - год через `VehicleAge::inYears` (`min_year`, `max_age_years`);
   - пробег `0..max_mileage_km` (500 000);
   - стоимость `market_value > 0`;
   - сумма `amount.min..max` (50 000..2 000 000);
   - срок `term.min..max_months` (3..48).
   При ошибках — `ValidationException`. Возвращает `{vin, year, mileage, market_value, requested_amount, term_months}`.
2. `LtvCalculator::calculate(requested_amount, market_value)` — `round(amount / market_value * 100, 2)`, проценты.
3. `DecisionEngine::decide(ltv)` — пороги из конфига:
   - `ltv < 60` → `approve`;
   - `60 <= ltv <= 85` → `review` (в коде первое сравнение строгое `<`, в докблоке — `<=`);
   - `ltv > 85` → `reject`.
4. Результат: `vehicle_age`, `ltv`, `decision`, `approved_limit` (запрошенная сумма при approve, иначе 0), `input`.

Лимит по `ltv_by_age` из конфига не считается — задача LOAN-12, не сделана.

## Куда встанет правило «пробег > 400 000 км → review»

Логичное место — `DecisionEngine::decide()`, но сейчас он принимает только `float $ltv` и про пробег не знает. Варианты:

- передать `mileage` в `DecisionEngine` (новый аргумент `decide()` или поле конструктора) и добавить проверку;
- либо в `AssessmentService::assess()` между расчётом LTV и решением: «если `mileage > 400000` и решение `approve` — понижаем до `review`».

### Что уже есть из входных данных

- `mileage` есть в нормализованном результате `ApplicationValidator::validate()` (строка 78) и в `$input`, доступном в `AssessmentService::assess()`.
- Паттерн «пороги в конфиге»: в `rules.php` есть секция `vehicle`, куда можно добавить, например, `max_mileage_for_approve => 400000`.

### Чего не хватает

- Передачи `mileage` в `DecisionEngine`: конструктор принимает только `{approve_max, review_max}`, `decide()` — только `float $ltv`.
- Самого порога 400 000 в `rules.php`.
- Логики понижения решения (approve → review) — нигде нет.

## Что уже проверяется про пробег

`ApplicationValidator::validate()`, строки 43–46: `int`-каст (по умолчанию `-1`, отсутствие поля ловится), проверка `0 <= mileage <= 500000` (`vehicle.max_mileage_km`). Это валидация, не решение: пробег вне диапазона отбрасывает заявку целиком через `ValidationException`, а не переводит в `review`. На решение approve/review/reject пробег сейчас не влияет — такой связи нет.