---
layout: post
title: "inheritance diamond problem (python)"
author: "twistfatezz"
date: 2023-04-07 21:57
categories: "2023"
tags:
  - python
---

### # what is a "diamond problem"?
```txt
                   ┌──────────┐                     $1 the TemporaryEmployee inherited 2 path from the
                   │ Employee │                        base class Employee, lead to diamond problem.
                   └─────┬────┘ 
          ┌──────────────+───────────────┐          $2 the __init__ of Tmp class need be revised to 
 ┌────────┴────────┐                     │             avoid invoking it from the unexpected path.
 │ SarlaryEmployee │                     │          
 └────────┬────────┘             ┌───────┴────────┐ $3 other common methods of the two path should
          │                      │ hourlyEmployee │    also be revised && compulsively chosen 1 path up,
   ┌──────┴─────┐                └───────┬────────┘    print mro might help.
   │ Secretrary │                        │
   └──────┬─────┘                        │
          └──────────────+───────────────┘
               ┌─────────┴─────────┐
               │ TemporaryEmployee │
               └───────────────────┘
```

<hr>

### # code example to illustrate diamond problem
common base class
```python
class Employee:
    def __init__(self, id, name):
        self.id = id
        self.name = name
```

inherited subclasses
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

class Secretary(SalaryEmployee):
    def work(self, hours):
        print(f'{self.name} expends {hours} hours doing office paperwork.')
```

invoker class
```python
class PayrollSystem:
    def calculate_payroll(self, employees):
        print('Calculating Payroll')
        print('===================')
        # calling method/attributes
        for employee in employees:
            print(f'Payroll for: {employee.id} - {employee.name}')
            print(f'- Check amount: {employee.calculate_payroll()}')
            print('')
```
<br>

4 version of implementations of successor classes <br>
```python
# version-1: 
# inherited order: Secretary -> HourlyEmployee
class TemporarySecretary(Secretary, HourlyEmployee):
    pass
```
```txt
# output:
Traceback (most recent call last):
temporary_secretary = TemporarySecretary(5, 'Robin Williams', 40, 9)
TypeError: __init__() takes 4 positional arguments but 5 were given
# mro: 
<class '__main__.TemporarySecretary'>, <class '__main__.Secretary'>, <class '__main__.SalaryEmployee'>, 
<class '__main__.HourlyEmployee'>, <class '__main__.Employee'>, <class 'object'>
# reason:
the main function is intend to use init of HourlyEmployee
according to the mro: the TemporarySecretary class use init of SalaryEmployee to instantiate 
```

```python
# version-2: 
# change the inheritance order: HourlyEmployee -> Secretary
class TemporarySecretary(HourlyEmployee, Secretary):
    pass
```
```txt
# output:
Traceback (most recent call last):
temporary_secretary = TemporarySecretary(5, 'Robin Williams', 40, 9)
super().__init__(id, name)
TypeError: __init__() missing 1 required positional argument: 'weekly_salary
# mro:
<class '__main__.TemporarySecretary'>, <class '__main__.HourlyEmployee'>, <class '__main__.Secretary'>, 
<class '__main__.SalaryEmployee'>, <class '__main__.Employee'>, <class 'object'>
# reason:
the main function is intend to use init of HourlyEmployee
w.r.t mro: the TemporarySecretary class use both the init of SalaryEmployee and HourlyEmployee
```

```python
# version-3: 
# inherited order: Secretary -> HourlyEmployee
# implement init by compulsively invoking HourlyEmployee's init
class TemporarySecretary(Secretary, HourlyEmployee):
    def __init__(self, id, name, hours_worked, hour_rate):
        HourlyEmployee.__init__(self, id, name, hours_worked, hour_rate)
```
```txt
# output:
Traceback (most recent call last):
in ps.calculate_payroll(company_employees)
in print(f'- Check amount: {employee.calculate_payroll()}')
in calculate_payroll return self.weekly_salary
AttributeError: 'TemporarySecretary' object has no attribute 'weekly_salary'
# mro: 
<class '__main__.TemporarySecretary'>, <class '__main__.Secretary'>, <class '__main__.SalaryEmployee'>, 
<class '__main__.HourlyEmployee'>, <class '__main__.Employee'>, <class 'object'>
# reason:
the main function is intend to use init of HourlyEmployee
w.r.t mro: the temporarySecretary class compulsively invoke HourlyEmployee init
the init is fine but ps.callculate_payroll is still invoking the method of SalaryEmployee
```

```python
# version-4: 
# inherited order: Secretary -> HourlyEmployee
# implement init by compulsively invoking HourlyEmployee's init
# overwrite calculate_payroll to compulsively appoints which one to invoke
class TemporarySecretary(Secretary, HourlyEmployee):
    def __init__(self, id, name, hours_worked, hour_rate):
        HourlyEmployee.__init__(self, id, name, hours_worked, hour_rate)

    def calculate_payroll(self):
        return HourlyEmployee.calculate_payroll(self)
```
```txt
# output:
Calculating Payroll
===================
Payroll for: 2 - John Smith
- Check amount: 1500

Payroll for: 5 - Robin Williams
- Check amount: 360 
```
<br>

main function to instantiate the diamond successor class "TemporarySecretary", and call the ambiguous common method "calculate_payroll"
```python
if __name__ == '__main__':
    print(TemporarySecretary.__mro__)

    secretary = Secretary(2, 'John Smith', 1500)
    temporary_secretary = TemporarySecretary(5, 'Robin Williams', 40, 9)
    company_employees = [secretary, temporary_secretary]

    ps = PayrollSystem()
    ps.calculate_payroll(company_employees)
```
<br>




