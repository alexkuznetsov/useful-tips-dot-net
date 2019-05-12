[How to indentify type of image when we have byte array of image](https://social.msdn.microsoft.com/Forums/vstudio/en-US/9d25b9b9-fd3d-4a1a-86b7-b5cd08089802/how-to-indentify-type-of-image-when-we-have-byte-array-of-image?forum=netfxbcl)
=
Question from Microsoft's forum
------------
Hi ,

Is there any way to indentify Image Type if we have byte array of image.

Thanks
Sachin Pithode


Answer
--------------

Hi, you have to read the header information of image stream to check its type.
the following code will read image header information.

```cs
        public static string GetImageType(string path)
        {
            string headerCode = GetHeaderInfo(path).ToUpper();

            if (headerCode.StartsWith("FFD8FFE0"))
            {
                return "JPG";
            }
            else if (headerCode.StartsWith("49492A"))
            {
                return "TIFF";
            }
            else if (headerCode.StartsWith("424D"))
            {
                return "BMP";
            }
            else if (headerCode.StartsWith("474946"))
            {
                return "GIF";
            }
            else if (headerCode.StartsWith("89504E470D0A1A0A"))
            {
                return "PNG";
            }
            else
            {
                return ""; //UnKnown
            }
        }

        public static string GetHeaderInfo(string path)
        {
            byte[] buffer = new byte[8];

            BinaryReader reader = new BinaryReader(new FileStream(path, FileMode.Open));
            reader.Read(buffer, 0, buffer.Length);
            reader.Close();

            StringBuilder sb = new StringBuilder();
            foreach (byte b in buffer)
                sb.Append(b.ToString("X2"));

            return sb.ToString();
        }
```

to call this function

```cs
string filePath = @"D:\test.jpg";

Console.WriteLine(GetImageType(filePath));
```

i hope this will help.
