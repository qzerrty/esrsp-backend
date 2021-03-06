# Создание

## Создание базы
```sql
CREATE DATABASE esrsp_db;
```

## Создание пользователя
```sql
create user esrsp_user with encrypted password 'POSTGRES_PASS';

grant all privileges on database esrsp_db to esrsp_user;
```

## Создание таблиц

### Преподаватель
```sql
CREATE TABLE Teacher
(
	Id SERIAL PRIMARY KEY,
	FullName VARCHAR(128) NOT NULL,
	Email VARCHAR(128) NOT NULL
);
```

### Дисциплина
```sql
CREATE TABLE Discipline
(
	Id SERIAL PRIMARY KEY,
	Name VARCHAR(128) UNIQUE
);
```

### Группа
```sql
CREATE TABLE StudentGroup
(
	Id SERIAL PRIMARY KEY,
	Name VARCHAR(16) UNIQUE
);
```

### Студент
```sql
CREATE TABLE Student
(
	Id SERIAL PRIMARY KEY,
	FullName VARCHAR(128) NOT NULL,
	RecordBook VARCHAR(16) UNIQUE,
	StudentGroupId INT,
	FOREIGN KEY (StudentGroupId) REFERENCES StudentGroup (Id) ON DELETE SET NULL
);
```

### Поток
```sql
CREATE TABLE Flow
(
	Id INT,
	StudentGroupId INT,
	PRIMARY KEY (Id, StudentGroupId),
	FOREIGN KEY (StudentGroupId) REFERENCES StudentGroup (Id) ON DELETE CASCADE
);
```

### Занятие
```sql
CREATE TABLE Class
(
	Id SERIAL PRIMARY KEY,
	FlowId INT NOT NULL,
	TeacherId INT NOT NULL,
	DisciplineId INT NOT NULL,
	ClassNumber SMALLINT NOT NULL,
	ClassDay SMALLINT NOT NULL,
	Place VARCHAR(16) NOT NULL,
	FOREIGN KEY (TeacherId) REFERENCES Teacher (Id) ON DELETE CASCADE,
	FOREIGN KEY (DisciplineId) REFERENCES Discipline (Id) ON DELETE CASCADE
);
```

### Успеваемость студента
```sql
CREATE TABLE AcademicPerformance
(
	Id SERIAL PRIMARY KEY,
	StudentId INT NOT NULL,
	DisciplineId INT NOT NULL,
	FOREIGN KEY (StudentId) REFERENCES Student (Id) ON DELETE CASCADE,
	FOREIGN KEY (DisciplineId) REFERENCES Discipline (Id) ON DELETE CASCADE
);
```

### Успеваемость в дату
```sql
CREATE TABLE AcademicPerformanceDate
(
	Id SERIAL PRIMARY KEY,
	AcademicPerformanceId INT NOT NULL,
	Date DATE NOT NULL,
	AcademicPerformance VARCHAR(64) NOT NULL,
	FOREIGN KEY (AcademicPerformanceId) REFERENCES AcademicPerformance (Id) ON DELETE CASCADE
);
```

### Аккаунты
```sql
create table Account
(
	Login VARCHAR(64) PRIMARY KEY,
	Password VARCHAR(64) NOT NULL,
	UserType INT DEFAULT 0,
	UserId INT NOT NULL
);
```
UserType:
0 - teacher
1 - teacher-admin
В планах добавление 2 - student

UserId - id учителя из таблицы teacher. Не связан через FOREIGN KEY, т.к. планируется добавление аккаунтов студентов

# Запросы
Частые запросы:
- расписание на две недели
- список группы с успеваемостью ее студентов по конкретному предмету
- список групп преподавателя

Запросы средней редкости:
- добавление мероприятия
- добавление оценки

Редкие запросы:
- добавление группы
- добавление расписания

## Добавление данных
Добавление преподавателя
```sql
INSERT INTO Teacher(FullName, Email) VALUES ('Дмитриев А.П.', 'dmitriev_ap@voenmeh.ru');
```

Добавление преподавателя-представителя деканата
```sql
INSERT INTO Teacher(FullName, Email, IsAdmin) VALUES ('Дмитриев А.П.', 'dmitriev_ap@voenmeh.ru', true);
```

Добавление дисциплины
```sql
INSERT INTO Discipline(Name) VALUES ('пр МЕТ.КРОСС.ТРАНСЛ.');
```

Добавление группы
```sql
INSERT INTO StudentGroup(Name) VALUES ('И595');
```

Добавление студента
```sql
INSERT INTO Student(FullName, RecordBook, StudentGroupId) VALUES ('Иванов И.И.', 'И59501', 1);
```

Добавление потока
```sql
INSERT INTO Flow(Id, StudentGroupId) VALUES (1, 1);
```

Добавление занятия
```sql
INSERT INTO Class(FlowId, TeacherId, DisciplineId, ClassNumber, ClassDay, Place) VALUES (1, 1, 1, 1, 1, '218*');
```

Добавление успеваемости студента
```sql
INSERT INTO AcademicPerformance(StudentId, DisciplineId) VALUES (1, 1);
```

Добавление успеваемости в дату
```sql
INSERT INTO AcademicPerformanceDate(AcademicPerformanceId, Date, AcademicPerformance) VALUES (1, 'YYYY-MM-DD', '5');
```

Генерация успеваемости студентов
```sql
insert into AcademicPerformance(StudentId, DisciplineId)
select distinct S.id, D.id from class as C
join discipline as D on D.id = C.disciplineid
right join flow as F on F.id = C.flowid
join studentgroup as SG on F.studentgroupid = SG.id
right join student as S on SG.id = S.studentgroupid
order by S.id, D.id;
```

## Получение

Получение списка групп-предметов преподавателя, отсортированного по id дисциплин, групп
```sql
select distinct
	D.name as discipline,
	D.id as disciplineid,
	SG.name as groupname,
	SG.id as groupid
from class as C
join discipline as D on D.id = C.disciplineid
right join flow as F on F.id = C.flowid
join studentgroup as SG on F.studentgroupid = SG.id
where teacherid = 3
order by D.id, SG.id;
```

Получение расписания группы на две недели
```sql
select
	D.name as discipline,
	T.fullname as teacher,
	(
		select STRING_AGG(SG.name, ', ') from studentgroup as SG
		where id in (
			select studentgroupid from flow
			where id = C.flowid
		)
	) as groups,
	C.place,
	C.classday,
	C.classnumber
from class as C
join discipline as D on D.id = C.disciplineid
join teacher as T on T.id = C.teacherid
where flowid in (
	select id from flow
	where studentgroupid = 1
)
order by C.classday, C.classnumber;
```

Получение расписания преподавателя на две недели
```sql
select
	D.name as discipline,
	T.fullname as teacher,
	(
		select STRING_AGG(SG.name, ', ') from studentgroup as SG
		where id in (
			select studentgroupid from flow
			where id = C.flowid
		)
	) as groups,
	C.place,
	C.classday,
	C.classnumber
from class as C
join discipline as D on D.id = C.disciplineid
join teacher as T on T.id = C.teacherid
where teacherid = 1
order by C.classday, C.classnumber;
```

Получение списка группы
```sql
select * from student
where studentgroupid = 1
order by fullname;
```

Получение списка дисциплин группы
```sql
select * from discipline
where id in (
	select disciplineid from class
	where flowid in (
		select id from flow
		where studentgroupid = 1
	)
) order by id;
```

Получение списка дисциплин студента
```sql
select * from discipline
where id in (
	select disciplineid from class
	where flowid in (
		select id from flow
		where studentgroupid = (
			select studentgroupid from student
			where id = 1
		)
	)
) order by id;
```

Получение названий группы и дисциплины
```sql
select
	SG.name as groupname,
	D.name as discipline
from studentgroup as SG, discipline as D
where SG.id = 1 and D.id = 1;
```