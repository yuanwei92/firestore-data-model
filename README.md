# Firestore Data Model
I have been developing with Vue.js and Firebase for a few years. Starting with Realtime Database and now I mostly use Firestore as my database. One of the thing that I find challenging is to come up with the right data model in Firestore for different use cases. Here I listed out some of the models I learned, put it down as a framework for myself to choose the right model. Do let me know if there are better solutions!

## Many-to-many Relationship
As far as I am aware, these are the three common ways to model many-to-many relationship in Firestore. In following examples I will use Students and Courses data, a student can join multiple courses and a course could have multiple students.

### Using Subcollections
Cons
- Need to update both studentCourses and courseStudents
- Cannot perform query (unless embed students/courses details into subcollection documents)
- If embed details in subcollection documents, need to perform update when the main document updated

```
students
  studentId
    studentCourses
      courseId

courses
  couresId
    courseStudents
      studentId
```

Getting all students who joined courseId1
```javascript
const studentsQuerySnapshot = await coursesRef.doc("courseId1").collection("courseStudents").get();
const studentDocSnapshots = await Promise.all(
  studentsQuerySnapshot.docs.map(snapshot => studentsRef.doc(snapshot.id))
)
const students = studentDocSnapshots.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads: 2N

Getting all courses joined by studentId1
```javascript
const coursesQuerySnapshot = await studentsRef.doc("studentId1").collection("studentCourses").get();
const courseDocSnapshots = await Promise.all(
  coursesQuerySnapshot.docs.map(snapshot => coursesRef.doc(snapshot.id))
)
const students = courseDocSnapshots.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads: 2N


### Using Maps/Array
Cons
- Firestore document size has a 1 MiB limit
- Not able to query one of the collection 
(Embeding maps in both documents could solve the issue but then you will need to update both of them when create/delete)

#### Maps
```
students
  studentId: {
    courses: {
      courseId1: timestamp,
      courseId2: timestamp,
    }
  }

courses
  courseId
```

Getting all students who joined courseId1
```javascript
let studentsQuerySnapshot = await studentsRef.where("courses.courseId", "!=", false).get();
let students = studentsQuerySnapshot.docs.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads : N


Getting all courses joined by studentId1
```javascript
const student = await studentsRef.doc("studentId1").get().then(snapshot => {
  const student = snapshot.data();
  student.id = snapshot.id;
  return student;
});
const courseIds = Object.keys(student.courses);
const courseSnapshots = await Promise.all(
  courseIds.map(courseId => coursesRef.doc(courseId).get())
)
const courses = courseSnapshots.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads : N

#### Array
```
students
  studentId: {
    courses: ["courseId1", "courseId2"]
  }

courses
  courseId
```

Getting all students who joined courseId1
```javascript
let studentsQuerySnapshot = studentsRef.where("courses", "array-contains", "courseId1").get();
let students = studentsQuerySnapshot.docs.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads : N

Getting all courses joined by studentId1
```javascript
const student = await studentsRef.doc("studentId1").get().then(snapshot => {
  const student = snapshot.data();
  student.id = snapshot.id;
  return student;
});
const courseIds = Object.keys(student.courses);
const courseSnapshots = await Promise.all(
  courseIds.map(courseId => coursesRef.doc(courseId).get())
)
const courses = courseSnapshots.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads : N

### Junction Table
Cons
- Cannot perform query
```
students
  studentId

courses
  courseId

courseStudents
  courseId_studentId: {
    courseId: "courseId1",
    studentId: "studentId1"
  }
```

Getting all students who joined courseId1
```javascript
const junctionQuerySnapshot = await courseStudentsRef.where("courseStudents", "studentId", "studentId1").get();
const courseSnapshots = await Promise.all(
  junctionQuerySnapshot.docs.map(snapshot => coursesRef.doc(snapshot.courseId).get())
)
const courses = courseSnapshots.map(snapshot => {
  id: snapshot.id,
  ...snapshot.data()
})
```
Document reads : 2N
