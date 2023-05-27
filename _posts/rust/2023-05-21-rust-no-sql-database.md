---
layout: post
title:  "Build a Simple Key Value Database from scratch with Rust"
date:   "2023-05-21 12:37:00 +0100"
tags: rust nosql
---

I've always been fascitaned by databases and always wanted to learn more about its internals, in this tutorial, we will build a simple Key-Value store from scratch with Rust. 

> Due to the length of this topic, we will split it into chapter and at the end of each chapter will be the link to the PR containing the change for that chaper.

Our Key-Value database is going to be called giraffeDB, why ? cos i love giraffes

Before we go ahead, it's important to know that our goal is to aim for simplicity, so a lot of the implementations will be simplified to make it easier to follow. 

## Storage

Our data is going to be stored to disk and it's important to understand how our data will be stored and retrieved. 

Databases typically store data in `pages` on disk, where a page is a fixed length contiguous block of memory. Pages are the smallest unit of data exchanged by the database and the disk. Database pages are stored contiguously on the disk to minimize disk seeks. [Here's more on pages](https://en.wikipedia.org/wiki/Page_(computer_memory))

At a high level, our database is just a binary file, could be 4MB or 4GB, but we will design it to store the data in pages, each of them with a number, so we can easily read from a page and write to a page.

Our Database is gonna have a page size of 4KB,unlike postgres that has one of [8KB](https://www.postgresql.org/docs/current/storage-page-layout.html),  which thus means our database can read 4KB of information at a go, , To  understand this better, let's imagine we want to store a collection of students with their images as keys and values like such.

| student id      | student score |
| ----------- | ----------- |
| 1      | 20       |
| 2   | 30        |
| 3   | 10        |
| 4   | 40        |

Let's now imagine that each row is the same as our page size (4KB), let's see how this will be arranged in memory.

<---- Insert Diagram of pages here ----->


So if we start a database with a 4GB size, this should mean, we can store 1,000 pages (4KB * 1000). 


Enough talk, let's write some code.

Before we start, we will break the code into multiple parts, that explains different building blocks of the DB, first we start with the Data Access Layer

## Part 1: Data Access Layer

Our data access layer will hold our database file and handle reading and writing from and to pages. It is the wrapper around our database file, and every read/write operation must be done via our dal. 

Our Dal will contain some basic configuration like page size, although, we'll fix it to 4KB, this is just so we can tweak it later on.

Let's create a new rust module called `dal.rs` where we'll store our Dal struct.

Here's what our dal initially looks like.

```rust

pub struct Dal {
    file : File,
    pageSize : u32
}

impl Dal{
    pub fn new(path : &str, pageSize : u32) -> Self {
        let file =  std::fs::OpenOptions::new()
            .create(true)
            .write(true)
            .append(true)
            .mode(0o666)
            .open(path)
            .unwrap();

        Dal {
            file,
            pageSize
        }
    }
}

```

Next thing is to create our Page struct, it should have a page number and hold data that is 4KB (Page Size for our DB).

```rust
pub struct Page {
    pub pageNumber : u64,
    pub data : Vec<u8>
}

impl  Page {
    pub fn new(pageNumber : Option<u64>) -> Self {
        Page {
            pageNumber: pageNumber.unwrap_or(0), // if no page number, set to 0
            data: vec![0_u8;  PageSize as usize] // create a 4kB zero filled vector 
        }
    }
}
```

Now we write helper methods in our Dal struct to read from pages, and write to pages, to do that first, we add this helper method to allocate an empty page.

```rust
pub fn allocateEmptyPage(&self, pageNumber : Option<u64>) -> Page {
    Page::new(pageNumber)
}
```

Next, we write methods for reading and writing pages. To read a page, we need the page number, remember our database file is just a binary file that's internally represented as a contiguous chunk of 4KB pages,so if we want to read page 5, we need to start reading from offset 20KB (5 * 4KB) and that will be the beginning of our page.

```rust
pub fn readPage(&self, pageNumber : u64) -> Page {
    let mut page = self.allocateEmptyPage(Some(pageNumber));
    let offset = pageNumber * self.pageSize as u64;

    // fill in the data from poisition 'offset' into the buffer page.data
    self.file.read_at(&mut page.data, offset).unwrap();

    page
}

pub fn writePage(&mut self, page : &mut Page) {
    // get the right offset for the page so as not to overwrite any other data
    let offset = page.pageNumber * self.pageSize as u64;

    // write at offset 'offset', all that's in the buffer 'page.data' 
    self.file.write_at(&page.data, offset).unwrap();
}

```

With this basic set up, we can test that this works by attempting to write to some pages, and then try to see if what we did works.

In our `main.rs` file, we here's a simple test we can run:

```rust
fn main() {
    // initialize Dal
    let mut dal = Dal::new("giraffe.db", dal::PageSize);

    let mut page_0 = dal.allocateEmptyPage(Some(0));

    writeToPage(&mut page_0.data, "Page 0, hello", 0);

    // commit page
    dal.writePage(&mut page_0);

    let mut page_1 = dal.allocateEmptyPage(Some(1));

    writeToPage(&mut page_1.data, "Page 1, hello", 0);

    // commit page
    dal.writePage(&mut page_1);
}

```

where the simple helper method `writeToPage` is defined as:

```rust
fn writeToPage(buffer: &mut[u8],msg : &str, startAt: usize) {
    // write to buffer from position `startAt`
    buffer[startAt .. msg.len()].copy_from_slice(msg.as_bytes());
}
```

Let's run our code with cargo run:
```bash
cargo run
```

Once this is done, hopefully there shouldn't be any errors and we should see the file `giraffe.db`. Now, that's a binary file and the best way to see what's inside it is with the tool `hexdump`

If we run the `hexdump` command with the `-C` flag, we can see ASCII representation of the bytes on the right as shown below

```bash
hexdump -C giraffe.db
```

we get this printed to our console.

![](/assets/images/hexdump-part-1.png)

We can easily see that the first page, represented by `00000000` to `00001000` just contains bytes that when interpreted as ASCII, reads `Page 0, hello` as seen on the right. The next 4KB, starting at `00001000` read `Page 1, hello`.

This a very good starting point, we have a Dal struct that can read and write a page, and we can see from our database file that it does as expected and our database can actually write to pages.

In the next part, we will introduce some new concepts that will improve our database. 

[Here's the github change](https://github.com/samuelorji/giraffeDB/pull/1/files)


## Part 2: FreeList, Metadata  and Persistence
### Freelist


As with any database, data often gets delete, what happens to the a page when all its data is deleted, do we clean it up, do we ignore it and see our database file continuously increase, or do we reuse those pages for newer data. 

We will use a Freelist to handle this issue, it is going to be responsoble for reusing/reclaiming empty pages. It's going to be a simple component that we'll attach to our Dal, it's going to hold two fields for now, once called `maxPage` which represents the maximum page number of our database currently, and a list of page numbers called `releasedPages` that will store pages that have been emptied and are available for reuse. 

So when we want to allocate a new page, the freelist first gives us a page from the `releasedPages` and it's empty, it increments the `maxPage` counter:

>We will do a little refactor here, we will make page numbers a 16 bit value as we don't expect a page number to be greater than a 16 bit value. So head on to the page struct and change the `pageNumber` field to be a 16 bit value 


```rust

#[derive(Debug)]
struct FreeList {
    maxPage : u16, // Holds the maximum page allocated. maxPage*PageSize = fileSize
    releasedPages: Vec<u16> // Pages that were previously allocated but are now free
}

impl FreeList {
    pub fn new() -> Self {
        Self {
            maxPage : 0,
            releasedPages: vec![]
        }
    }

    pub fn getNextPageNumber(&mut self) -> u16 {
        // if possible, get pages from released pages first, else increment maxPage and return it
        if let Some(releasedPageId) = self.releasedPages.pop() {
            releasedPageId
        } else {
            self.maxPage += 1 ;
            self.maxPage
        }
    }

    pub fn releasePage(&mut self, pageNumber : u16) {
        self.releasedPages.push(pageNumber)
    }
}

```

we can then add this as a component to the dal struct when creating it.

```rust
impl Dal {
    pub fn new(path : &str, pageSize : u32) -> Self {

         // rest of previous code // 
        let freeList = FreeList::new();

        Dal {
            file,
            pageSize,
            freeList
        }
    }
}

```

### Metadata

It's typical for databases to store some metadata in a page, this typically holds data that's crucial for the database operation. For now, we will just have store the freelist page.

This metadata will always be stored in Page 0, the very first page and it will also be part of the Dal component. 

We will add more metadata as we continue. Let's create a struct that we'll call `Meta`


```rust
#[derive(Debug)]
struct Meta {
    freeListPage : u16
}

impl Meta {
    pub  fn new() -> Self {
        Self {
            // start with a seed value of 0, this may change later on
            freeListPage : 0
        }
    }
}

```

Now, let's add this to our dal component.

```rust
impl Dal {
    pub fn new(path : &str, pageSize : u32) -> Self {

         // rest of previous code // 
         let meta = Meta::new();

        Dal {
            file,
            pageSize,
            freeList,
            meta
        }
    }
}

```


### Persistence

If you've noticed, you'll notice that our database seems to be recreated everytime the application is restarted. That is definitely not good, and in this section, we will try to implement persistence for our database.

Basically we want a way to serialize and deserialize our Dal to and from disk. To do this, we need to be able to convert our `FreeList` and `Meta` data structyres to and from bytes. 

Since we're going to be doing some bytes manipulation, let's use the [bytes](https://docs.rs/bytes/1.4.0/bytes/) crate to help with this ([installation here](https://docs.rs/bytes/1.4.0/bytes/)).

> I'm using a little endian machine, [here's how to find out the endianness of your machine](https://stackoverflow.com/a/66308651)

To formalize serialization and deserialization of structs to bytes, we can define the following traits in a new module called `util.rs`. This module may hold utilities like constants.

```rust
pub trait Serialize<T> {
    fn serialize(&self, buffer : &mut [u8]);
}

pub trait Deserialize<T> {
    fn deserialize(&mut self, buffer : &[u8]);
}
```

Our `serialize` function, takes a mutable buffer and serializes `self` into the buffer, while the `deserialize` function takes a mutable `self`and does the opposite.

Let's now create serialize and deserialize function for our `Meta` and `FreeList` structs:

```rust
impl Serialize<Meta> for Meta {
    fn serialize(&self, buffer: &mut [u8]) {
        // To serialize a Meta
        // - 8 bytes for the freelist page
        let mut position: usize = 0;
        LittleEndian::write_u64(&mut buffer[position..], self.freeListPage);
        position += 8;
    }
}

impl Deserialize<Meta> for Meta {
    fn deserialize(&mut self, buffer: &[u8]) {
        let mut position: usize = 0;
        // read 8 bytes for freelist position at position `position`
        let freeListPage = LittleEndian::read_u64(&buffer[position..]);
        position += PAGE_NUM_SIZE as usize;

        self.freeListPage = freeListPage
    }
}
```

For the FreeList:

```rust
impl Serialize<FreeList> for FreeList {
    fn serialize(&self, buffer: &mut [u8]) {

        // To serialize a Meta
        // - 2 bytes for max page (u16)
        // - 2 bytes for length of released list (vector in our case)
        // - 2 bytes for each released page number

        // write max page (2 bytes)
        let mut cursor: u64 = 0;
        LittleEndian::write_u16(&mut buffer[cursor as usize..], self.maxPage);
        cursor += 2;

        // write length of released pages (2 bytes)
        LittleEndian::write_u16(&mut buffer[cursor as usize..], self.releasedPages.len() as u16);
        cursor += 2;

        // for each released page, write the released page number (2 bytes)
        for pgNumber in &self.releasedPages {
            LittleEndian::write_u16(&mut buffer[cursor as usize..], *pgNumber);
            cursor += 2;
        }
    }
}

impl Deserialize<FreeList> for FreeList {
    fn deserialize(&mut self, buffer: &[u8]) {

        // To deserialize a Meta
        // - 2 bytes for max page (u16)
        // - 2 bytes for length of released list (vector in our case)
        // - 2 bytes for each released page number

        // read max page (2 bytes)
        let mut cursor : usize = 0;
        let maxPage = LittleEndian::read_u16(&buffer);
        cursor +=2;

        // read length of released pages (2 bytes)
        let numReleasedPages = LittleEndian::read_u16(&buffer[cursor .. ]);
        cursor +=2;

        let mut releasedPages : Vec<u16> = Vec::with_capacity(numReleasedPages as usize);

        // for each released page, read the released page number (2 bytes)
        for _ in 0..numReleasedPages {
            let releasedPage = LittleEndian::read_u16(&buffer[cursor as usize ..]);
            releasedPages.push(releasedPage);
            cursor += 2

        }

        self.maxPage = maxPage ;
        self.releasedPages = releasedPages
    }
}

```

Now that we've defined `serialize` and `deserialize` methods for our structs, let's write some helper methods on the dal struct that can read and write a meta and freelist:

```rust

impl Dal {
    // rest of code
    pub fn writeMeta(&mut self) -> Page {
        // meta page is 0 
        let mut page =  self.allocateEmptyPage(Some(META_PAGE_NUMBER)); // META_PAGE_NUMBER = 0
        
        // serialize the meta struct into the `page.data` buffer
        self.meta.serialize(&mut page.data[..]);

        // write the page to disk
        self.writePage(&mut page);

        page
    }

    pub fn readMeta(&mut self) -> Meta {
        // read page
        let page = self.readPage(META_PAGE_NUMBER);

        // create an empty meta
        let mut meta = Meta::new();

        // deserialize what's in the read `page.data` into the empty meta struct
        meta.deserialize(&page.data);
        meta
    }

    pub fn writeFreeList(&mut self) -> Page {
        // get freelist page from meta
        let freeListPage = self.meta.freeListPage;
        
        let mut page = self.allocateEmptyPage(Some(freeListPage));
        
        // serialize free list into the empty page
        self.freeList.serialize(&mut page.data);

        // write page 
        self.writePage(&mut page);
        page
    }

    pub fn readFreeList(&mut self) {
        // get freelist page from meta
        let freeListPage = self.meta.freeListPage;
        
        // read freelist page
        let mut page = self.readPage(freeListPage);

        // deserialize free list page
        self.freeList.deserialize(&mut page.data);

    }
}

```

One other thing we need to do is change how the `Dal::new()` method that creates a new `Dal` works, currently it just assumes that the file doesn't exist and probably tries to recreate it. This is definitely not what we want. What we want instead is for the method to check if the database file exists, if it does, it tries to deserialize some of the data in it like the `Meta` and `FreeList`, otherwise it creates the new file. 

> We will do some refactoring here to not accept page size from outside also 

Here's what the method will look like now:

```rust
impl Dal {

    pub fn new(path : &str) -> Self {
        if(!Path::new(path).exists()) {
            // database file doesn't exist, create a new database file
            let file =  std::fs::OpenOptions::new()
                .create(true)
                .write(true)
                .read(true)
                .append(true)
                .mode(0o666)
                .open(path)
                .unwrap();

            c
            let mut freeList = FreeList::new();
            let mut meta = Meta::new();


            let mut dal = Dal {
                file,
                pageSize : PAGE_SIZE, // PAGE_SIZE is 4KB (4096)
                meta,
                freeList
            };

            // set freelist to page 1 and increment maxPage in the freelist struct
            let freeListPageNumber = dal.freeList.getNextPageNumber();
            dal.meta.freeListPage = freeListPageNumber;

            // write the meta struct to disk
            dal.writeMeta();
            dal

        } else {
            // file exists, read meta to find freelist page and deserialize accordingly
            let file = std::fs::OpenOptions::new()
                .write(true)
                .read(true)
                .append(true)
                .open(path)
                .unwrap();

            // create new freelist and mets struct
            let mut freeList = FreeList::new();
            let mut meta = Meta::new();

            let mut dal = Dal {
                file,
                pageSize : PAGE_SIZE,
                meta,
                freeList
            };

            // read meta from disk and store in dal struct
             dal.readMeta();

            // read freeList from and store in dal struct
            dal.readFreeList();
            dal
        }
    }

    // rest of code
}

```



