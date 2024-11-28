
针对Excel下打开表格图片显示 \#NAME？编辑栏显示为 \=@\_xlfn.DISPIMG( 样公式的问题，一般需要在 wps 程序下，Ctrl\+F 查找范围选值，输入 \=DISPIMG 全选，然后再右键转换为浮动图片。如果是Excel中，则是查找公式 DISPIMG。
查阅网上得资料得知，可以通过解压表格文件，然后 根据公式中的第一参数（通常以 ID 开头）看 xl\_rels\\cellimages.xml.rels 目录下的 name （其值为 dispimg 函数的第一参数）和 r:embed （其值以 rId 开头）的对应关系，然后再看 Id （ rId 开头 ） 和 Target（图片路径） 的对应关系，进而得到图片的路径。
在 LLM 的帮助下进而有以下 PS 脚本。



```
function Get-ExcelDispImages {
    param (
        [Parameter(Mandatory=$true)]
        [string]$ExcelPath,
        
        [Parameter(Mandatory=$false)]
        [string]$OutputFolder = ".\ExcelImages"
    )

    # 辅助函数：安全地读取文件内容
    function Read-FileContent {
        param (
            [string]$Path
        )
        try {
            # 使用.NET方法直接读取文件，避免PowerShell路径解析问题
            if ([System.IO.File]::Exists($Path)) {
                return [System.IO.File]::ReadAllText($Path)
            }
            Write-Host "File not found: $Path" -ForegroundColor Yellow
            return $null
        }
        catch {
            Write-Host "Error reading file $Path : $_" -ForegroundColor Yellow
            return $null
        }
    }

    try {
        # 验证Excel文件是否存在
        if (-not (Test-Path -LiteralPath $ExcelPath)) {
            throw "Excel file not found: $ExcelPath"
        }

        # 确保ExcelPath是绝对路径
        $ExcelPath = (Get-Item $ExcelPath).FullName

        # 创建输出文件夹（使用绝对路径）
        $OutputFolder = [System.IO.Path]::GetFullPath($OutputFolder)
        if (-not (Test-Path -Path $OutputFolder)) {
            New-Item -ItemType Directory -Path $OutputFolder -Force | Out-Null
        }

        # 创建Excel COM对象
        $excel = New-Object -ComObject Excel.Application
        $excel.Visible = $false
        $excel.DisplayAlerts = $false
        $workbook = $excel.Workbooks.Open($ExcelPath)
        
        # 用于存储找到的DISPIMG ID和信息
        $dispImgIds = @()
        $imageMapping = @{}

        # 遍历所有工作表
        foreach ($worksheet in $workbook.Worksheets) {
            $usedRange = $worksheet.UsedRange
            $rowCount = $usedRange.Rows.Count
            $colCount = $usedRange.Columns.Count
            
            for ($row = 1; $row -le $rowCount; $row++) {
                for ($col = 1; $col -le $colCount; $col++) {
                    $cell = $usedRange.Cells($row, $col)
                    $formula = $cell.Formula
                    
                    # 检查是否包含DISPIMG函数并提取所有参数
                    if ($formula -match 'DISPIMG\("([^"]+)"') {
                        $imageId = $matches[1]
                        Write-Host "Found DISPIMG: $formula" -ForegroundColor Gray
                        
                        # 创建参数列表
                        $params = @{
                            'ID' = $imageId
                            'Cell' = $cell.Address()
                            'Formula' = $formula
                            'Worksheet' = $worksheet.Name
                        }
                        
                        # 提取所有参数，包括图片ID
                        $paramValues = @()
                        $formula -match 'DISPIMG\((.*?)\)' | Out-Null
                        $paramString = $matches[1]
                        $paramValues = $paramString -split ',' | ForEach-Object { 
                            $_ -replace '"', '' -replace '^\s+|\s+$', ''
                        }
                        
                        # 存储所有参数
                        for ($i = 0; $i -lt $paramValues.Count; $i++) {
                            $params["Param$i"] = $paramValues[$i]
                        }
                        
                        $imageMapping[$imageId] = $params
                        $dispImgIds += $imageId
                    }
                }
            }
        }

        # 创建临时目录
        $tempPath = Join-Path ([System.IO.Path]::GetTempPath()) "ExcelTemp"
        if (Test-Path $tempPath) {
            Remove-Item $tempPath -Recurse -Force
        }
        New-Item -ItemType Directory -Path $tempPath -Force | Out-Null

        # 复制Excel文件到临时目录并解压
        $tempExcel = Join-Path $tempPath "temp.xlsx"
        $tempZip = Join-Path $tempPath "temp.zip"
        Copy-Item -Path $ExcelPath -Destination $tempExcel -Force
        if (Test-Path $tempZip) { Remove-Item $tempZip -Force }
        Rename-Item -Path $tempExcel -NewName "temp.zip" -Force
        Expand-Archive -Path $tempZip -DestinationPath $tempPath -Force

        # 检查media文件夹并处理图片
        $mediaPath = Join-Path $tempPath "xl\media"
        if (Test-Path $mediaPath) {
            # 显示DISPIMG参数和图片对应关系
            Write-Host "`nFound $($imageMapping.Count) DISPIMG functions" -ForegroundColor Cyan
            
            if ($imageMapping.Count -gt 0) {
                Write-Host "`n=== DISPIMG Functions Found ===" -ForegroundColor Cyan
                Write-Host "Found $($imageMapping.Count) DISPIMG functions" -ForegroundColor Cyan
                
                # 显示所有找到的DISPIMG函数
                Write-Host "`n=== DISPIMG Functions Details ===" -ForegroundColor Yellow
                foreach ($id in $imageMapping.Keys) {
                    $params = $imageMapping[$id]
                    Write-Host "Cell: [$($params.Worksheet)]$($params.Cell)" -ForegroundColor Gray
                    Write-Host "Formula: $($params.Formula)" -ForegroundColor Gray
                }
                
                # 首先从cellimages.xml获取DISPIMG ID到rId的映射
                $dispImgToRid = @{}
                $cellImagesPath = Join-Path $tempPath "xl\cellimages.xml"
                Write-Host "`n=== Reading cellimages.xml ===" -ForegroundColor Yellow
                Write-Host "Path: $cellImagesPath" -ForegroundColor Gray
                
                if (Test-Path $cellImagesPath) {
                    try {
                        $xmlContent = Get-Content $cellImagesPath -Raw -EnCoding UTF8
                        Write-Host "`nRaw XML Content:" -ForegroundColor Gray
                        Write-Host $xmlContent
                        
                        # 使用正则表达式提取所有cellImage元素
                        $matches = [regex]::Matches($xmlContent, '.*?', [System.Text.RegularExpressions.RegexOptions]::Singleline)
                        Write-Host "`nFound $($matches.Count) cellImage elements" -ForegroundColor Gray
                        
                        foreach ($match in $matches) {
                            $cellImageXml = $match.Value
                            Write-Host "`nProcessing cellImage element:" -ForegroundColor Gray
                            
                            # 提取name属性（包含DISPIMG ID）
                            if ($cellImageXml -match 'name="([^"]+)"') {
                                $dispImgId = $matches[1]
                                Write-Host "Found DISPIMG ID: $dispImgId" -ForegroundColor Gray
                                
                                # 提取r:embed属性（包含rId）
                                if ($cellImageXml -match 'r:embed="(rId\d+)"') {
                                    $rId = $matches[1]
                                    $dispImgToRid[$dispImgId] = $rId
                                    Write-Host "  Mapping: DISPIMG ID $dispImgId -> rId $rId" -ForegroundColor Green
                                }
                            }
                        }
                    }
                    catch {
                        Write-Host "Error reading cellimages.xml: $($_.Exception.Message)" -ForegroundColor Red
                        Write-Host $_.Exception.StackTrace -ForegroundColor Red
                    }
                }
                else {
                    Write-Host "cellimages.xml not found!" -ForegroundColor Red
                }
                
                # 从cellimages.xml.rels获取rId到实际图片的映射
                $ridToImage = @{}
                $cellImagesRelsPath = Join-Path $tempPath "xl\_rels\cellimages.xml.rels"
                Write-Host "`n=== Reading cellimages.xml.rels ===" -ForegroundColor Yellow
                Write-Host "Path: $cellImagesRelsPath" -ForegroundColor Gray
                
                if (Test-Path $cellImagesRelsPath) {
                    try {
                        [xml]$relsXml = Get-Content $cellImagesRelsPath -Raw
                        Write-Host "`nXML Content:" -ForegroundColor Gray
                        Write-Host $relsXml.OuterXml
                        
                        $relsXml.Relationships.Relationship | ForEach-Object {
                            if ($_.Target -match "media/") {
                                $ridToImage[$_.Id] = $_.Target
                                Write-Host "  rId $($_.Id) -> $($_.Target)" -ForegroundColor Green
                            }
                        }
                    }
                    catch {
                        Write-Host "Error reading cellimages.xml.rels: $($_.Exception.Message)" -ForegroundColor Red
                        Write-Host $_.Exception.StackTrace -ForegroundColor Red
                    }
                }
                else {
                    Write-Host "cellimages.xml.rels not found!" -ForegroundColor Red
                }
                
                Write-Host "`n=== Summary of Mappings ===" -ForegroundColor Yellow
                Write-Host "DISPIMG ID -> rId mappings:" -ForegroundColor Gray
                $dispImgToRid.GetEnumerator() | ForEach-Object {
                    Write-Host "  $($_.Key) -> $($_.Value)" -ForegroundColor Gray
                }
                
                Write-Host "`nrId -> Image mappings:" -ForegroundColor Gray
                $ridToImage.GetEnumerator() | ForEach-Object {
                    Write-Host "  $($_.Key) -> $($_.Value)" -ForegroundColor Gray
                }
                
                # 处理每个DISPIMG函数
                Write-Host "`n=== Processing DISPIMG Functions ===" -ForegroundColor Yellow
                foreach ($id in $imageMapping.Keys) {
                    $params = $imageMapping[$id]
                    
                    Write-Host "`nProcessing: [$($params.Worksheet)]$($params.Cell)" -ForegroundColor Green
                    Write-Host "Formula: $($params.Formula)" -ForegroundColor Gray
                    Write-Host "DISPIMG ID: $id" -ForegroundColor Gray
                    
                    # 从DISPIMG ID查找对应的rId
                    $rId = $dispImgToRid[$id]
                    if ($rId) {
                        Write-Host "Found rId: $rId" -ForegroundColor Green
                        
                        # 从rId查找对应的图片路径
                        $imagePath = $ridToImage[$rId]
                        if ($imagePath) {
                            Write-Host "Found image path: $imagePath" -ForegroundColor Green
                            $imageName = Split-Path $imagePath -Leaf
                            
                            # 查找对应的图片文件
                            $mediaFile = Get-ChildItem -LiteralPath $mediaPath | Where-Object { $_.Name -eq $imageName }
                            if ($mediaFile) {
                                $outputPath = Join-Path $OutputFolder "$id$($mediaFile.Extension)"
                                Copy-Item -LiteralPath $mediaFile.FullName -Destination $outputPath -Force
                                Write-Host "Successfully copied: $imageName -> $outputPath" -ForegroundColor Cyan
                           #    以 media 文件夹下的文件名复制
                           #    Copy-Item  $mediaFile.FullName  $(Join-Path $OutputFolder $(Split-Path $imagePath -Leaf))
                            }

                            else {
                                Write-Host "Image file not found: $imageName" -ForegroundColor Red
                            }
                        }
                        else {
                            Write-Host "No image path found for rId: $rId" -ForegroundColor Red
                        }
                    }
                    else {
                        Write-Host "No rId found for DISPIMG ID: $id" -ForegroundColor Red
                    }
                }
            } else {
                Write-Host "No DISPIMG functions found in the workbook." -ForegroundColor Yellow
            }
        }
    }
    catch {
        Write-Host "错误: $($_.Exception.Message)" -ForegroundColor Red
    }
    finally {
        # 清理
        if ($workbook) {
            $workbook.Close($false)
        }
        if ($excel) {
            $excel.Quit()
            [System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null
        }
        if (Test-Path $tempPath) {
            Remove-Item $tempPath -Recurse -Force
        }
        [System.GC]::Collect()
        [System.GC]::WaitForPendingFinalizers()
    }
}

```

一个可能的用法如下
Get\-ExcelDispImages \-ExcelPath "C:\\Users\\demo\\Documents\\dispimg\_wps.xlsx" \-OutputFolder $pwd\\output
生成的图片在当前目录下的 output 文件夹下。


参考阅读：


 本博客参考[西部世界官网](https://www.xbsj9.com)。转载请注明出处！
