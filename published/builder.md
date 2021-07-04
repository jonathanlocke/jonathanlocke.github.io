
#### <img src="https://state-of-the-art.org/graphics/kivakit/kivakit-32.png" srcset="https://state-of-the-art.org/graphics/kivakit/kivakit-32-2x.png 2x" style="vertical-align:middle"/> &nbsp; [2021.07.04 - A (very) slightly simpler builder pattern](#builder)  

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />
<a name = "builder"></a>

2021.07.04

### A (very) slightly simpler builder pattern &nbsp; <img src="https://state-of-the-art.org/graphics/wand/wand-32.png" srcset="https://state-of-the-art.org/graphics/wand/wand-32-2x.png 2x" style="vertical-align:baseline"/>

[Frank Kiwy](https://blogs.oracle.com/author/frank-kiwy) recently published, in Java Magazine, the article [*Exploring Joshua Bloch’s Builder design pattern in Java: Bloch’s Builder pattern can be thought of as a workaround for a missing language feature*](https://blogs.oracle.com/javamagazine/java-builder-pattern-bloch). I have a fine point to make about it. There is a way to eliminate a bit of repetition from the example in this article, making the code a little easier to read and more succinct. The example looks something like this (I've reduced the number of fields and changed the formatting, because I'm just like that):

    public class Book 
    {
        private final String isbn;
        private final String title;
        private final String author;
        
        private Book(Builder builder) 
        {
            this.isbn = builder.isbn;
            this.title = builder.title;
            this.author = builder.author;
        }
    
        public String getIsbn() 
        {
            return isbn;
        }
    
        public String getTitle() 
        {
            return title;
        }
    
        public String getAuthor() 
        {
            return author;
        }
        
        public static class Builder 
        {
            private final String isbn;
            private final String title;
            private String author;
    
            public Builder(String isbn, String title) 
            {
                this.isbn = isbn;
                this.title = title;
            }
    
            public Builder author(String author) 
            {
                this.author = author;
                return this;
            }
    
            public Book build() 
            {
                return new Book(this);
            }
        }
    }

My quibble is that we can get rid of the duplicate fields in the *Builder* class above by instead using an instance of *Book*. The difference between the two is that my version is a little bit simpler and clearer, and the line count drops from 52 lines in Josh's builder pattern to 47 in mine (and the more fields there are, the more this difference will widen). It is true that in my version, the fields in *Book* are no longer explicitly declared as *final*. However, this makes no difference because the fields can't be mutated except by *Builder*. They are effectively immutable once *build()* is called. 

    public class Book 
    {
        private String isbn;
        private String title;
        private String author;
        
        private Book() 
        {
        }
    
        public String getIsbn() 
        {
            return isbn;
        }
    
        public String getTitle() 
        {
            return title;
        }
    
        public String getAuthor() 
        {
            return author;
        }
        
        public static class Builder 
        {
            private Book book = new Book();
    
            public Builder(String isbn, String title) 
            {
                book.isbn = isbn;
                book.title = title;
            }
    
            public Builder author(String author) 
            {
                book.author = author;
                return this;
            }
    
            public Book build() 
            {
                return book;
            }
        }
    }
     
Questions? Comments? Tweet yours to @OpenKivaKit.
