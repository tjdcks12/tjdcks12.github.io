---
title: Deeeep Sequelize!
excerpt: |
  Node js 의 ORM Sequelize
categories:
  - ORM
feature_image: "/assets/banners/banner1.png"
---

## Sequelize



Sequelize는 node js에서 지원하는 ORM이다.



Spring JPA를 활용하면서 ORM이 주는 편리함에 대해서 굉장히 만족스러웠기 때문에, 

새로 진행될 node 프로젝트 내에서도 ORM을 사용해보겠다는 생각을 했고, 이를 통해 선택한 도구가 바로 Sequelize 였다.



ORM은 기본적으로 Object mapping 개념이 포함되기 때문에 자바스크립트내의 자료형을 구현한 Typescript를 기반으로 활용하면 좋을 것이라는 생각에, 자연스럽게 Typescript 기반의 Sequelize 활용기가 되겠다.



등장하는 예제 코드는 [Sequelize 공식문서]([https://sequelize.org](https://sequelize.org/)) 를 참고하였다.



### 모델 정의 (Model Definition)

ORM에서 가장 기본이 되는 Model 정의이다.

Sequelize에서는 모델을 정의하기 위한 함수로 define이라는 함수를 제공한다.



```javascript
sequelize.define('Foo', {
  firstname: Sequelize.STRING,
  lastname: Sequelize.STRING
}, {
  getterMethods: {
    fullName() {
      return this.firstname + ' ' + this.lastname;
    }
  },

  setterMethods: {
    fullName(value) {
      const names = value.split(' ');

      this.setDataValue('firstname', names.slice(0, -1).join(' '));
      this.setDataValue('lastname', names.slice(-1).join(' '));
    }
  }
});
```



다만 타입스크립트를 통해 define 메소드를 활용하게 될경우, ORM의 도메인객체자체를 호출하기 위한 형태 (class 혹은 interface)로 표현하는것에 제약이 생기게 된다.



때문에 이부분을 class로 구현하였고, 그 형태는 다음과 같이 구현가능하다.

```javascript
class Foo extends Model {
  get fullName() {
    return this.firstname + ' ' + this.lastname;
  }

  set fullName(value) {
    const names = value.split(' ');
    this.setDataValue('firstname', names.slice(0, -1).join(' '));
    this.setDataValue('lastname', names.slice(-1).join(' '));
  }
}
Foo.init({
  firstname: Sequelize.STRING,
  lastname: Sequelize.STRING
}, {
  sequelize,
  modelName: 'foo'
});
```





이렇게 정의한 클래스를 Dao 단에서 메소드화 시킨 형태는 다음과 같다.

```javascript
Employee
  .create({ name: 'John Doe', title: 'senior engineer' })
  .then(employee => {
    console.log(employee.get('name')); // John Doe (SENIOR ENGINEER)
    console.log(employee.get('title')); // SENIOR ENGINEER
  })
```





### 유효성 검사 (Validation)

객체에 적절한 값이 매핑되어 동작하는것을 보장하는 validation 또한 지원한다.

필드 네임에 객체 블록을 생성하고 validate 객체블록을 통해 적용 가능하며, 다양한 옵션이 존재하므로 이는 공식문서를 참조하면 될듯하다.



```javascript
class ValidateMe extends Model {}
ValidateMe.init({
  bar: {
    type: Sequelize.STRING,
    validate: {
      is: ["^[a-z]+$",'i'],     // will only allow letters
      is: /^[a-z]+$/i,          // same as the previous example using real RegExp
      not: ["[a-z]",'i'],       // will not allow letters
      isEmail: true,            // checks for email format (foo@bar.com)
      isUrl: true,              // checks for url format (http://foo.com)
      isIP: true,               // checks for IPv4 (129.89.23.1) or IPv6 format
      isIPv4: true,             // checks for IPv4 (129.89.23.1)
      isIPv6: true,             // checks for IPv6 format
      isAlpha: true,            // will only allow letters
      isAlphanumeric: true,     // will only allow alphanumeric characters, so "_abc" will fail
      isNumeric: true,          // will only allow numbers
      isInt: true,              // checks for valid integers
      isFloat: true,            // checks for valid floating point numbers
      isDecimal: true,          // checks for any numbers
      isLowercase: true,        // checks for lowercase
      isUppercase: true,        // checks for uppercase
      notNull: true,            // won't allow null
      isNull: true,             // only allows null
      notEmpty: true,           // don't allow empty strings
      equals: 'specific value', // only allow a specific value
      contains: 'foo',          // force specific substrings
      notIn: [['foo', 'bar']],  // check the value is not one of these
      isIn: [['foo', 'bar']],   // check the value is one of these
      notContains: 'bar',       // don't allow specific substrings
      len: [2,10],              // only allow values with length between 2 and 10
      isUUID: 4,                // only allow uuids
      isDate: true,             // only allow date strings
      isAfter: "2011-11-05",    // only allow date strings after a specific date
      isBefore: "2011-11-05",   // only allow date strings before a specific date
      max: 23,                  // only allow values <= 23
      min: 23,                  // only allow values >= 23
      isCreditCard: true,       // check for valid credit card numbers

      // Examples of custom validators:
      isEven(value) {
        if (parseInt(value) % 2 !== 0) {
          throw new Error('Only even values are allowed!');
        }
      }
      isGreaterThanOtherField(value) {
        if (parseInt(value) <= parseInt(this.otherField)) {
          throw new Error('Bar must be greater than otherField.');
        }
      }
    }
  }
}, { sequelize });
```



다음과같이 필드 구성에 따른 병렬적 유효성 검사도 가능하므로, 활용도가 굉장히 높은편이다.

```javascript
class Pub extends Model {}
Pub.init({
  name: { type: Sequelize.STRING },
  address: { type: Sequelize.STRING },
  latitude: {
    type: Sequelize.INTEGER,
    allowNull: true,
    defaultValue: null,
    validate: { min: -90, max: 90 }
  },
  longitude: {
    type: Sequelize.INTEGER,
    allowNull: true,
    defaultValue: null,
    validate: { min: -180, max: 180 }
  },
}, {
  validate: {
    bothCoordsOrNone() {
      if ((this.latitude === null) !== (this.longitude === null)) {
        throw new Error('Require either both latitude and longitude or neither')
      }
    }
  },
  sequelize,
})
```



### 설정 (Configuration)

Sequelize에서는 자체 설정이 굉장히 많은편이다. 스프링 JPA에서는 일일히 설정해주어야 했던 사소한 기능들 (예를들면, table naming strategy, table name mapping, createdAt, updatedAt과 같은 Auditing 과 관련해서)이 이미 설정되있는 경우가 많다. 때문에 이러한 부분들을 직접 컨트롤하기 위해서는 **configuration 설정정보**를 변경해주어야한다.



configuration은 모델 정의를 위한 init 함수의 두번째 인자 (options) 를 통해 주입 가능하다. 

이또한 예시로 살펴보도록 하겠다.



```javascript
class Foo extends Model {}
Foo.init({ /* bla */ }, {
  // don't forget to enable timestamps!
  timestamps: true,

  // I don't want createdAt
  createdAt: false,

  // I want updatedAt to actually be called updateTimestamp
  updatedAt: 'updateTimestamp',

  // And deletedAt to be called destroyTime (remember to enable paranoid for this to work)
  deletedAt: 'destroyTime',
  paranoid: true,

  sequelize,
})
```





### 다른 모델 주입 (Import)

ORM에서 빠지지 않는 설정중 하나가 다른 모델과 관련한 연관 관계이다. 

기본적으로 Sequelize에서는 다른 모델 정보를 가져오기 위해 import 함수를 이용한다.

**JS 기본문법과는 구분해야한다.** (Sequelize 라이브러리 내부적으로 캐싱되는 기능적 요소또한 존재한다고 하니, 도큐먼테이션에서 권장하는 방식을 사용하자)



```javascript
const Project = sequelize.import(__dirname + "/path/to/models/project")

// The model definition is done in /path/to/models/project.js
// As you might notice, the DataTypes are the very same as explained above
module.exports = (sequelize, DataTypes) => {
  class Project extends sequelize.Model { }
  Project.init({
    name: DataTypes.STRING,
    description: DataTypes.TEXT
  }, { sequelize });
  return Project;
}
```

 



### Optimistic Lock

configuration에서 version 프로퍼티를 true로 설정할경우 version에 따른 트랙잭션 검사를 통해, 다른 버전간의 갱신이 이루어질경우 OptimisticLockError를 발생시킨다.



### 데이터베이스 동기화 (Database synchronization)

DDL 설정과 같은 부분이다. 모델 정의부에 지정한 객체정보대로 데이터베이스를 생성할수 있는 함수들이 존재하며, 다음과 같이 사용한다.

```javascript
// Create the tables:
Project.sync()
Task.sync()

// Force the creation!
Project.sync({force: true}) // this will drop the table first and re-create it afterwards

// drop the tables:
Project.drop()
Task.drop()

// event handling:
Project.[sync|drop]().then(() => {
  // ok ... everything is nice!
}).catch(error => {
  // oooh, did you enter wrong database credentials?
})
```

Because synchronizing and dropping all of your tables might be a lot of lines to write, you can also let Sequelize do the work for you:

```javascript
// Sync all models that aren't already in the database
sequelize.sync()

// Force sync all models
sequelize.sync({force: true})

// Drop all tables
sequelize.drop()

// emit handling:
sequelize.[sync|drop]().then(() => {
  // woot woot
}).catch(error => {
  // whooops
})
```





### 메소드 (Method)

자바의 static과같이 사용되는 클래스레벨 메소드, 일반 메소드와 같이 사용되는 인스턴스레벨 메소드 두가지 형태의 메소드가 존재한다.



```javascript
class User extends Model {
  // Adding a class level method
  static classLevelMethod() {
    return 'foo';
  }

  // Adding an instance level method
  instanceLevelMethod() {
    return 'bar';
  }
}
User.init({ firstname: Sequelize.STRING }, { sequelize });
```





### 인덱싱 (indexs)

인덱싱 설정을 위한 옵션도 존재한다. 이는 데이터베이스 싱크 Model.sync(), sequelize.sync() 과정에서 동작한다.



```javascript
class User extends Model {}
User.init({}, {
  indexes: [
    // Create a unique index on email
    {
      unique: true,
      fields: ['email']
    },

    // Creates a gin index on data with the jsonb_path_ops operator
    {
      fields: ['data'],
      using: 'gin',
      operator: 'jsonb_path_ops'
    },

    // By default index name will be [table]_[fields]
    // Creates a multi column partial index
    {
      name: 'public_by_author',
      fields: ['author', 'status'],
      where: {
        status: 'public'
      }
    },

    // A BTREE index with an ordered field
    {
      name: 'title_index',
      using: 'BTREE',
      fields: ['author', {attribute: 'title', collate: 'en_US', order: 'DESC', length: 5}]
    }
  ],
  sequelize
});
```