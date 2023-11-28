---
layout: post
title: "inheritance && duck typing (python)"
author: "twistfatezz"
date: 2023-04-06 22:06
categories: "2023"
tags:
  - python
---

### # duck typing
In weak type language, any object that implements the desired interface can be used in place of another object. 
that is: “if it behaves like a duck, then it’s a duck.”

<hr>

### # inheritance graph
```txt
               abstract class (ABC)
                ┌──────────┐
                │ Employee │
                └─────┬────┘
           ┌──────────┴───────────┐             absolote class (to validate duck typing)
    ┌──────┴───────┐       ┌──────┴───────┐  ┌───────────────────┐
    │SalaryEmployee+--+    │HourlyEmployee│  │DisgruntledEmployee│
    └──────┬───────┘  |    └──────+───────┘  └────────+──────────┘
┌──────────┴───────┐  |           |                   |
│CommissionEmployee│  +--------+  |  +----------------+
└──────────+───────┘           |  |  |
           |               +---+--+--+---+
           +---------------+PayrollSystem| invoker class
                           +-------------+
```

<hr>

### # code sample to illustrate inheritance && duck typing feature
abstract base class (ABC)
```python
class Employee:
    def __init__(self, id, name):
        self.id = id
        self.name = name
```

inherited class
```python
class SalaryEmployee(Employee):
    def __init__(self, id, name, weekly_salary):
        super().__init__(id, name)
        self.weekly_salary = weekly_salary

    def calculate_payroll(self):
        return self.weekly_salary

class HourlyEmployee(Employee):
    def __init__(self, id, name, hours_worked, hour_rate):
        super().__init__(id, name)
        self.hours_worked = hours_worked
        self.hour_rate = hour_rate

    def calculate_payroll(self):
        return self.hours_worked * self.hour_rate

class CommissionEmployee(SalaryEmployee):
    def __init__(self, id, name, weekly_salary, commission):
        super().__init__(id, name, weekly_salary)
        self.commission = commission

    def calculate_payroll(self):
        fixed = super().calculate_payroll()
        return fixed + self.commission
```

an extra class without correlation with base class employee
```python
class DisgruntledEmployee:
    def __init__(self, id, name):
        self.id = id
        self.name = name

    def calculate_payroll(self):
        return 1000000
```

class method caller
```python
class PayrollSystem:
    def calculate_payroll(self, employees):
        print('Calculating Payroll')
        print('===================')
        # calling obj's method
        for employee in employees:
            print(f'Payroll for: {employee.id} - {employee.name}')
            print(f'- Check amount: {employee.calculate_payroll()}')
            print('')
```

main function
```python
if __name__ == '__main__':
    # init inherited obj
    salary_employee = SalaryEmployee(1, 'John Smith', 1500)
    hourly_employee = HourlyEmployee(2, 'Jane Doe', 40, 15)
    commission_employee = CommissionEmployee(3, 'Kevin Bacon', 1000, 250)

    # init extra obj
    disgruntled_employee = DisgruntledEmployee(20000, 'Anonymous')

    # init processing obj 
    payroll_system = PayrollSystem()

    # calling obj.method w.r.t duck typing(weak type language):
    # any object that implements the desired interface can be used in place of another object. 
    # that is: “if it behaves like a duck, then it’s a duck.”
    payroll_system.calculate_payroll([salary_employee, hourly_employee, 
        commission_employee, disgruntled_employee])
```
```txt
Calculating Payroll
===================
Payroll for: 1 - John Smith
- Check amount: 1500

Payroll for: 2 - Jane Doe
- Check amount: 600

Payroll for: 3 - Kevin Bacon
- Check amount: 1250

Payroll for: 20000 - Anonymous
- Check amount: 1000000
```

