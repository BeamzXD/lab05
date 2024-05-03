## Answers Laboratory work V

## Homework

### Задание
1. Создайте `CMakeList.txt` для библиотеки *banking*.
2. Создайте модульные тесты на классы `Transaction` и `Account`.
    * Используйте mock-объекты.
    * Покрытие кода должно составлять 100%.
3. Настройте сборочную процедуру на **TravisCI**.
4. Настройте [Coveralls.io](https://coveralls.io/).

## Выполнение

1. Создадим файл СMakeList.txt в котором пропишим:
````
cmake_minimum_required(VERSION 3.4)
project(Test_banking)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)

if(BUILD_TESTS)
  add_compile_options(--coverage)
endif()

add_library(banking STATIC ${CMAKE_CURRENT_SOURCE_DIR}/banking/Transaction.cpp ${CMAKE_CURRENT_SOURCE_DIR}/banking/Account.cpp)
target_include_directories(banking PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/banking)
target_link_libraries(banking gcov)

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB BANKING_TEST_SOURCES tests/*.cpp)
  add_executable(check ${BANKING_TEST_SOURCES})
  target_link_libraries(check banking gtest_main gmock_main)
  add_test(NAME check COMMAND check)
endif()
````
Здесь прописано все то, что было уаказано в туториале к лабароторной, за исключением 8-10 и 14 строчки:

````
if(BUILD_TESTS)
  add_compile_options(--coverage)
endif()
...
````
Здесь, если опция BUILD_TESTS включена, эта часть кода добавляет флаг компилятора --coverage, который необходим для сбора информации о покрытии кода. 

````
target_link_libraries(banking gcov)
````

Данная строчка линкует нашу библеотеку banking c gcov, еобходимой для сбора информации о покрытии кода.

2. Перейдем к созданию модульных тестов:
Создадим в директории tests создадим файл tests.cpp в котором будут прописаны наши тесты, тесты будут проведени с помощью mock-объектов и без них. Итоговый вариант тестов выглядит так:
````
#include <gtest/gtest.h>
#include <gmock/gmock.h>

#include "Account.h"
#include "Transaction.h"

class AccountMock : public Account {
public:
	AccountMock(int id, int balance) : Account(id, balance) {}
	MOCK_CONST_METHOD0(GetBalance, int());
	MOCK_METHOD1(ChangeBalance, void(int diff));
	MOCK_METHOD0(Lock, void());
	MOCK_METHOD0(Unlock, void());
};
class TransactionMock : public Transaction {
public:
	MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Account, Mock) {
	AccountMock acc(1, 100);
	EXPECT_CALL(acc, GetBalance()).Times(1);
	EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
	EXPECT_CALL(acc, Lock()).Times(2);
	EXPECT_CALL(acc, Unlock()).Times(1);
	acc.GetBalance();
	acc.ChangeBalance(100); // throw
	acc.Lock();
	acc.ChangeBalance(100);
	acc.Lock(); // throw
	acc.Unlock();
}

TEST(Account, SimpleTest) {
	Account acc(1, 100);
	EXPECT_EQ(acc.id(), 1);
	EXPECT_EQ(acc.GetBalance(), 100);
	EXPECT_THROW(acc.ChangeBalance(100), std::runtime_error);
	EXPECT_NO_THROW(acc.Lock());
	acc.ChangeBalance(100);
	EXPECT_EQ(acc.GetBalance(), 200);
	EXPECT_THROW(acc.Lock(), std::runtime_error);
}

TEST(Transaction, Mock) {
	TransactionMock tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
	.Times(6);
	tr.set_fee(100);
	tr.Make(ac1, ac2, 199);
	tr.Make(ac2, ac1, 500);
	tr.Make(ac2, ac1, 300);
	tr.Make(ac1, ac1, 0); // throw
	tr.Make(ac1, ac2, -1); // throw
	tr.Make(ac1, ac2, 99); // throw
}

TEST(Transaction, SimpleTest) {
	Transaction tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	tr.set_fee(100);
	EXPECT_EQ(tr.fee(), 100);
	EXPECT_THROW(tr.Make(ac1, ac1, 0), std::logic_error);
	EXPECT_THROW(tr.Make(ac1, ac2, -1), std::invalid_argument);
	EXPECT_THROW(tr.Make(ac1, ac2, 99), std::logic_error);
	EXPECT_FALSE(tr.Make(ac1, ac2, 199));
	EXPECT_FALSE(tr.Make(ac2, ac1, 500));
	EXPECT_TRUE(tr.Make(ac2, ac1, 300));
}
````
В ходе тестов была обнаружена ошибка в исходных файлах проекта в методе Make класса Transaction. В файле Transaction.cpp в реализации метода Make в Debit передавался аккаунт того, куда отправлять. Таким образом деньги у первого просто не снимались, если операция вообще происходила.

3. Создадим сборку для github Actions аналогично прошлой лабораторной работе: создаем CI.yml. Пропишем в него сборку проекта с включением coveralls.io:
````
name: CMake

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build_Linux:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Adding gtest
      run: git clone https://github.com/google/googletest.git third-party/gtest -b release-1.11.0

    - name: Install lcov
      run: sudo apt-get install -y lcov

    - name: Config banking with tests
      run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON -DCMAKE_CXX_FLAGS='--coverage'

    - name: Build banking
      run: cmake --build ${{github.workspace}}/build

    - name: Run tests
      run: |
        build/check
        lcov --directory . --capture --output-file coverage.info
        lcov --remove coverage.info '/usr/*' --output-file coverage.info
        lcov --remove coverage.info '${{github.workspace}}/third-party/gtest/*' --output-file coverage.info
        lcov --list coverage.info

    - name: Coveralls
      uses: coverallsapp/github-action@master
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        path-to-lcov: ${{ github.workspace }}/coverage.info
````
Рассмотрим релизацию Coveralls.io в сборке:
````
- name: Install lcov
  run: sudo apt-get install -y lcov
````
В данной строчке мы устанавливаем утилиту lcov, которая необходима для сбора информации о покрытии кода. Дальше запускаем CMake для конфигурирования сборки проекта banking с включенными тестами и флагом --coverage(о котором мы говорили, смотря в файл CMakeLists.txt) для сбора информации о покрытии кода.
````
    - name: Config banking with tests
      run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON -DCMAKE_CXX_FLAGS='--coverage'
````
Теперь подробнее расмотрим шаг Run tests. Первая строчка запускает наши тесты. Дальше строчка 
````
lcov --directory . --capture --output-file coverage.info
````
Собирает информацию о покрытии кода с помощью lcov и сохраняет ее в файл coverage.info. Следующие две строчки
````
lcov --remove coverage.info '/usr/*' --output-file coverage.info
lcov --remove coverage.info '${{github.workspace}}/third-party/gtest/*' --output-file coverage.info
````
удаляют из файла coverage.info информацию о системных библиотеках и информацию о библиотеках Google Test. Это сделано для того чтобы мы смотрели покрытие кода только для наших файлов и библеотек, а не вмести со сторонним. И последняя строчка этого шага
````
lcov --list coverage.info
````
выводит содержимое файла coverage.info на консоль.

И конечный шаг это отправка все на Coveralls.io:
````
- name: Coveralls
  uses: coverallsapp/github-action@master
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    path-to-lcov: ${{ github.workspace }}/coverage.info
````

На этом шаге используем действие coverallsapp/github-action для отправки информации о покрытии кода, содержащейся в файле coverage.info, на сервис Coveralls.io. Для этого требуется токен GitHub (${{ secrets.GITHUB_TOKEN }}).

Вот результат покрытия кода:

[![Coverage Status](https://coveralls.io/repos/github/BeamzXD/lab05/badge.svg?branch=main)](https://coveralls.io/github/BeamzXD/lab05?branch=main)


```
Copyright (c) 2024 Lozanov Ilia IU8-25
```
