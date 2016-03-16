---
layout: post
category : lessons
tagline: "Part 1"
tags : [SOLID, repositories, interface]
---
{% include JB/setup %}

A few years ago, the company I was working for began to move from a legacy ORM to Entity Framework.  The way we'd been using the previous solution, pretty much everything wound up being very tightly coupled.  As a result, our "Unit Tests" usually ended up involving initializing a test database, reading in configuration files, and other things that we really shouldn't have been doing.  Most developers hated writing tests and not many were written.

But this time, we were going to "do it right".  As part of that initative, I was asked to reasearch the correct way to use the repository pattern with EF so that we could easily unit test the code and not wind up in the same situation.  "This should be easy", I thought, and I fired up Google.

...And it turns out it's not so easy.  If you search the web for ["EF repository pattern"](https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=ef%20repository%20pattern), you'll wind up with a lot of hits.  The problem that you'll quickly run into is that most of the results don't agree with each other!

A large percentage of what you'll find looks something like the following:

    public interface IStudentRepository : IDisposable
    {
        IEnumerable<Student> GetStudents();
        Student GetStudentByID(int studentId);
        void InsertStudent(Student student);
        void DeleteStudent(int studentID);
        void UpdateStudent(Student student);
        void Save();
    }

     public class StudentRepository : IStudentRepository, IDisposable
    {
        private SchoolContext context;

        public StudentRepository(SchoolContext context)
        {
            this.context = context;
        }

        public IEnumerable<Student> GetStudents()
        {
            return context.Students.ToList();
        }

        public Student GetStudentByID(int id)
        {
            return context.Students.Find(id);
        }

        public void InsertStudent(Student student)
        {
            context.Students.Add(student);
        }

        public void DeleteStudent(int studentID)
        {
            Student student = context.Students.Find(studentID);
            context.Students.Remove(student);
        }

        public void UpdateStudent(Student student)
        {
            context.Entry(student).State = EntityState.Modified;
        }

        public void Save()
        {
            context.SaveChanges();
        }

        private bool disposed = false;

        protected virtual void Dispose(bool disposing)
        {
            if (!this.disposed)
            {
                if (disposing)
                {
                    context.Dispose();
                }
            }
            this.disposed = true;
        }

        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this);
        }
    }

In fact, this is taken directly from [www.asp.net](http://www.asp.net/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application), so you'd assme this is probably the right way to do it, right?

So is this ok?  Well, this will give us the basics of what we need.  If we code around the interface correctly, we can replace the implementation with something else (like an in-memory collection) pretty easily.  The problem is if we use this, we're also losing a lot of the capabilites of EF in the abstraction.  What if we want the first 10 students with the last name "Smith" ordered by first name.  This is easy with EF:

    var selectedStudents = context.Students.Where(x => x.LastName == "Smith").OrderBy(x => x.FirstName).Take(10).ToList();

But there's no way to do this with the above class.

For this reason one of the other common variants of the EF Repository is the following:

    public interface IStudentRepository : IDisposable
    {
        IQueryable<Student> GetStudents();
        Student GetStudentByID(int studentId);
        void InsertStudent(Student student);
        void DeleteStudent(int studentID);
        void UpdateStudent(Student student);
        void Save();
    }

By exposing IQueryable instead of IEnumerable, we get much of the flexibility of Entity Framework back.  The problem is we've also made any unit test where we substitute an in-memory implementation for the EF implementation completely invalid.  Don't believe me?  Try running the following query against an in-memory collection, and then against an EF DbSet:

This works fine in the in-memory collection, but throws an exception when you try it with the EF implementation.  This is because when running against EF, the underlying implementation for the IQueryable<T> interface is completely different than what gets used when running queries against an in-memory collection.  

Since we're trying to come up with an abstraction for writing our tests against, this makes exposing the IQueryable part of the interface downright dangerous as it makes it possible for our tests to pass, but for the queries to then fail when we're actually running against a database with EF.  IQueryable is great at what it does, but you should [never try to use it as a point of abstraction](http://blog.ploeh.dk/2012/03/26/IQueryableTisTightCoupling/).

So do we need to give up then?  Is impossible to have both flexibility and a clean line of abstraction between database code and our business logic?  Tune in for Part2 to find out!
