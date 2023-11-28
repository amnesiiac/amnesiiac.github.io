---
layout: post
title: "composition (python)"
author: "twistfatezz"
date: 2023-04-08 12:38
categories: "2023"
tags:
  - python
---
### # what is a "composition"?
```text
definition composition: a class contains an object of another class.
1 the class can reuse/leverage the implementation of the components it contains.
2 relation between two classes is considered loosely coupled/more flexible (unlike inheritance):
    2.1 changes to the component class rarely affect the composite class, 
    2.2 and changes to the composite class never affect the component class.
```

### # code sample to illustrate composition
```python
# outer class
class Address:
    def __init__(self, street, city, state, zipcode, street2=''):
        self.street = street
        self.street2 = street2
        self.city = city
        self.state = state
        self.zipcode = zipcode

    # any class inplement __str__ method can use print()/str() to get output
    def __str__(self):
        lines = [self.street]
        if self.street2:
            lines.append(self.street2)
        lines.append(f"{self.city}, {self.state} {self.zipcode}")
        return ' '.join(lines)
```
```python
# composition class: integrate outer class Address obj as a member
class Employee:
    def __init__(self, id, name):
        self.id = id
        self.name = name
        self.address = None
```

```python
# inheritance classes
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

```python
# user class: calling method via class in composition relationship
class PayrollSystem:
    def calculate_payroll(self, employees):
        print('Calculating Payroll')
        print('===================')
        for employee in employees:
            print(f'Payroll for: {employee.id} - {employee.name}')
            print(f'- Check amount: {employee.calculate_payroll()}')
            if employee.address:
                print(f'- Sent to: {employee.address}')
            print('')
```
```python
# main
if __name__ == '__main__':
    salary_employee = SalaryEmployee(1, 'John Smith', 1500)
    salary_employee.address = Address('Ningqiao St.', 'Yunqiao St.', 'SH', '200030')
    hourly_employee = HourlyEmployee(2, 'Jane Doe', 40, 15)
    commission_employee = CommissionEmployee(3, 'Kevin Bacon', 1000, 250)
    commission_employee.address = Address('Guangfulin St.', 'Jinqiao St.', 'SH', '200130')

    payroll_system = PayrollSystem()
    payroll_system.calculate_payroll([salary_employee, hourly_employee, commission_employee
])
```
```text
# output:
Calculating Payroll
===================
Payroll for: 1 - John Smith
- Check amount: 1500
- Sent to: Ningqiao St. Yunqiao St., SH 200030

Payroll for: 2 - Jane Doe
- Check amount: 600

Payroll for: 3 - Kevin Bacon
- Check amount: 1250
- Sent to: Guangfulin St. Jinqiao St., SH 200130
```
