---
layout: single
title:  "Go - Archive files with archive/tar lib"
date:   2018-09-09 08:00:00 +0800
categories: Go
---
## 前言
由於專案要提供 API 來讓使用者 export 匯出檔案， 因此需要將所需檔案集結成一個 archive file。這個流程是透過 Go 的標準 lib `archive/tar` 來處理，以下文章將簡單介紹流程和實作方式，並於最後附上完整程式碼。

## Tar
`tar` 在UNIX/Linux系統中是最常見的打包工具，透過 `tar` 的協助，我們可以把數個檔案打包成一個 `<file name>.tar` 檔案，以利進行後續處理。

![Go-Tar-1.png]({{ site.url }}/assets/images/Go-Tar-1.png)

### tar file format
`.tar` 的檔案格式主要是由 `file object` 和其對應的 `header` 所組成。 `header` 裡面包含了一個 file 的 metadata（例如檔案名稱，數據大小等），這樣系統就可以透過 header 去檢測檔案屬性和完整度。

## Tar files with Go
![Go-Tar-2.png]({{ site.url }}/assets/images/Go-Tar-2.png)
### 1. Create a Tar.Writer
在打包檔案時，首要先做的就是建立一個 tar file 的 writer，後續才能將需要打包的檔案 file 和 header 寫入到 tar file 中。

```Go
// 建立一個 tar file 的檔案位置.
target := filepath.Join(tarPath, fmt.Sprintf("%s.tar", tarName))
// 根據上面所建立的檔案位置來創建檔案. (回傳值為 *io.File，實作了 io.writer interface)
tarfile, err := os.Create(target)
if err != nil {
	return err
}
// 建立一個內含 tar 檔案的 tar writer
tarWriter := tar.NewWriter(tarfile)
```

### 2.讀取要打包的檔案
接下來，我們要一一讀取需要打包的檔案們，以便後續將這些檔案寫入到 tar.Writer 中。透過標準函式庫 `filepath.Walk()` 就能輕鬆走訪各 folder 底下的檔案群。

```Go
source := "<source folder>"

// Check source is existing.
info, err := os.Stat(source)
if err != nil {
	return nil
}

filepath.Walk(source, func(path string, info os.FileInfo, err error) error {
  // Read each file
})

```

### 3. 將要打包的檔案寫入到 tar.Writer 中
上面有提到，在打包 file 時，需要包含其對應的 `header`，因此我們可以透過 `tar.FileInfoHeader` 的幫助來去擷取 `io.File` 內的資訊，然後得到我們所要的 `header`。

```Go
header, err := tar.FileInfoHeader(info, info.Name())
// func FileInfoHeader(fi os.FileInfo, link string) (*Header, error)
// 這邊的 link 是提供給 file mode 為 ModeSymlink 時使用。
// 詳細可以參考 https://en.wikipedia.org/wiki/Symbolic_link
```

```Go
// 這邊要特別注意的是， io.File Name 不包含路徑，所以如果你的檔案是放在其他子 folder 底下，要記得重新設定 `header.name`。如果沒有重新設定的話，就不會放到對應 folder 底下啦。
header.Name = filepath.Join(baseDir, strings.TrimPrefix(path, source))
// strings.TrimPrefix(path, source) 這個用意是為了去除 source path
```

有了 `header` 之後，就把它寫入 tar.Writer 吧！

```Go
if err := tarWriter.WriteHeader(header); err != nil {
	return err
}
```

寫入了 `header` 之後，再把 file 內容也寫進去。

```Go
// 打開檔案
file, err := os.Open(path)
if err != nil {
	return err
}
defer file.Close()
// 將檔案複製到 tar.Writer
_, err = io.Copy(tarWriter, file)
```

### 4. 關閉 tar.Writer 
當把所有檔案和 `header` 寫到 tar file 之後，就可以把 tar.Writer 和 file stream 關掉。

```Go
tarfile.Close()
tarWriter.Close()
```

## Source Code

```Go
// Archive Create a tar file with multiple resources
func Archive(sources []string, tarPath, tarName string) error {

	var err error
	// Create the tar path
	target := filepath.Join(tarPath, fmt.Sprintf("%s.tar", tarName))
	// Create the tar file
	tarfile, err := os.Create(target)
	if err != nil {
		return err
	}
	defer tarfile.Close()

	// Create a tar writer
	tarWriter := tar.NewWriter(tarfile)
	defer tarWriter.Close()

	for _, source := range sources {
		if err = tarFile(source, tarWriter); err != nil {
			return err
		}
	}
	return nil
}

func tarFile(source string, tarWriter *tar.Writer) error {
	// Check source is existing.
	info, err := os.Stat(source)
	if err != nil {
		return nil
	}

	var baseDir string
	if info.IsDir() {
		baseDir = filepath.Base(source)
	}
	return filepath.Walk(source,
		func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			// Create a tar header for a single file
			header, err := tar.FileInfoHeader(info, info.Name())
			if err != nil {
				return err
			}
			fmt.Printf("%#v \n", header)

			if baseDir != "" {
				// Name of file entry, a full path is needed.
				// strings.TrimPrefix - Remove unnecessarily path
				header.Name = filepath.Join(baseDir, strings.TrimPrefix(path, source))
			}

			if err := tarWriter.WriteHeader(header); err != nil {
				return err
			}

			// Write
			if info.IsDir() {
				return nil
			}

			file, err := os.Open(path)
			if err != nil {
				return err
			}
			defer file.Close()
			// io.Copy is copy the content of src file
			_, err = io.Copy(tarWriter, file)
			return err
		})
}
```