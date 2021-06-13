![cover](/assets/cover.png)

# Preface

Hi reader, I'm [Randy](https://github.com/djyde), a front-end software engineer. Thank you for your interest in this book.

I built some SaaS side projects in my spare time with Next.js. In my opinion, Next.js is a great framework for building fullstack web application. Because you don't need to care about the build config when you bootstrap a project, just write your React pages and the API routes, everything works fine. You can write frontend and backend code in a single place and they can even share a common TypeScript's type definition.

But the in first few times I use Next.js to build a web application, I'm not so happy. Because I'm lack of something like 'best practice'. Should I use `react-query` or `swr`? What is the best way to handle error in API route? How do I response a 403 error in my service(model) layer without touching the `res` object? How do I implement a login system in Next.js (Next.js even doesn't have a built-in helper for sending cookie)?

For me, I wish I could just focus on the main features when I start building a SaaS project, instead of considering these many 'how to'.

Fortunately, after building a few projects, I've summarized some best practices. These best practices and techniques allow me to complete my projects very quickly (and robust!). They just like a pattern, I use this pattern to finish my SaaS project over and over again.

After finishing [Cusdis](https://cusdis.com), I start thinking about sharing how do I use these pattern and techniques. I was going to write a series of blog posts, but then I thought why not just write a small book about it. This is how this book came to be.

## Who this book is for

This book is for the people who want to or who is building fullstack application with Next.js. If you haven't used Next.js building an entire application, you can see roughly how it is in this book. If you have already built some projects with Next.js, you can  still get some inspiration in this book, to make your code more reusable and make error handling more easy.

## What this book is not

This is NOT a book to teach you the basic of Next.js. I don't teach you what `getServerSideProps()` is, what `res.json()` means, etc. I assume you know the basic. I don't think you should learn these from a book, you should learn from the official documentation. The value of this book is the thing that's not in the documentation, which is from my experience.

## What this book includes?

This book includes some techniques that I usually use in my Next.js projects:

- [react-query](https://react-query.tanstack.com/) A data fetching solution for React
- [next-connect ](https://github.com/hoangvvo/next-connect) A library to make use of connect-middleware in Next.js
- [Prisma](https://prisma.io) A Node.js ORM

This book won't go into these library at length. (Again, you should learn the API from the documentation, not from a book). Instead, I will focus on telling you what are they, why use them and how to use them on your fullstack application.

The most important part of this book is the `Real world examples`. You can learn how to apply these techniques in a real word application, how they will help you write your application more quickly. The sections before `Real world examples` is only for highlighting some usage of these libraries that you should know before diving into the real world examples.

Finnaly, English is not my first language. If you find anywhere I didn't express clearly, please mention me at Twitter ([@randyloop](https://twitter.com/randyloop)). 