### 1.导入Excel

```
/// <summary>
/// 查询Excel数据
/// </summary>
/// <param name="fileUrl"></param>
/// <returns></returns>
public static DataTable GetExcelDatatable(string fileUrl)
{
    //支持.xls和.xlsx，即包括office2010等版本的   HDR=Yes代表第一行是标题，不是数据；
    //string cmdText = "Provider=Microsoft.ACE.OLEDB.12.0;Data Source={0};Extended Properties='Excel 12.0; HDR=Yes; IMEX=1'";
    string cmdText = "Provider=Microsoft.ACE.OLEDB.12.0;Data Source={0};Extended Properties=\"Excel 12.0;HDR=YES\"";
    System.Data.DataTable dt = null;
    //建立连接
    OleDbConnection conn = new OleDbConnection(string.Format(cmdText, fileUrl));
    try
    {
        //打开连接
        if (conn.State == ConnectionState.Broken || conn.State == ConnectionState.Closed)
        {
            conn.Open();
        }

        System.Data.DataTable schemaTable = conn.GetOleDbSchemaTable(OleDbSchemaGuid.Tables, null);
        string strSql = "select * from [Sheet1$]";
        OleDbDataAdapter da = new OleDbDataAdapter(strSql, conn);
        DataSet ds = new DataSet();
        da.Fill(ds);
        dt = ds.Tables[0];
        return dt;
    }
    catch (Exception exc)
    {
        throw exc;
    }
    finally
    {
        conn.Close();
        conn.Dispose();
    }
}

/// <summary>
/// 返回数据
/// </summary>
/// <returns></returns>
public static string ReturnData(HttpRequest request)
{
    string result = string.Empty;
    string dir = string.Empty;
    string uploadPath = string.Empty;
    string filename = string.Empty;
    string suffix = string.Empty;
    string fullname = string.Empty;
    //HttpRequest request = System.Web.HttpContext.Current.Request;
    HttpFileCollection fileCollection = request.Files;
    string brand = request.Form[0];
    string category = request.Form[1];
    int testPaperId = int.Parse(request.Form[3]);
    dir = "/upload/" + brand + "/" + category + "/" + DateTime.Today.ToString("yyyyMMdd") + "/";
    uploadPath = AppDomain.CurrentDomain.SetupInformation.ApplicationBase + dir;
    if (fileCollection.Count == 1)
    {
        try
        {
            HttpPostedFile httpPostedFile = fileCollection[0];
            filename = Path.GetFileNameWithoutExtension(httpPostedFile.FileName) + Guid.NewGuid().ToString();
            suffix = Path.GetExtension(httpPostedFile.FileName).ToString().ToLower(); ;
            if (suffix == ".xlsx" || suffix == ".xls")
            {
                if (!Directory.Exists(uploadPath))
                {
                    Directory.CreateDirectory(uploadPath);
                }
                fullname = filename + suffix;
                string filePath = uploadPath + fullname;
                while (File.Exists(filePath))
                {
                    filename = Path.GetFileNameWithoutExtension(httpPostedFile.FileName) + Guid.NewGuid().ToString();
                    fullname = filename + suffix;
                    filePath = uploadPath + fullname;
                }
                //string savePath = Server.MapPath(filePath);
                httpPostedFile.SaveAs(filePath);
                DataTable ds = new DataTable();
                ds = GetExcelDatatable(filePath);
                if (testPaperId > 0)
                {
                    result = UploadSubject(ds, brand, testPaperId);//导入考题
                }
                else
                {
                    result = UploadUser(ds, brand);//导入用户
                }
            }
            else
            {
                result = "只可以选择Excel文件";
            }
        }
        catch (Exception e)
        {
            result = e.ToString();
        }
    }
    else
    {
        result = "文件数量不对";
    }
    return result;
}

/// <summary>
/// 导入用户
/// </summary>
/// <param name="ds"></param>
/// <returns></returns>
public static string UploadUser(DataTable ds, string brand)
{
    string result = string.Empty;
    DataRow[] dr = ds.Select();            //定义一个DataRow数组
    int rowsnum = ds.Rows.Count;
    int successly = 0;
    if (rowsnum > 0)
    {
        string _Result = "";
        for (int i = 0; i < dr.Length; i++)
        {
            try
            {
                string username = dr[i]["姓名"].ToString();
                string mobile = dr[i]["手机号"].ToString();
                string province = dr[i]["省份"].ToString();
                string city = dr[i]["城市"].ToString();
                string depot = dr[i]["店铺"].ToString();
                int usertype = int.Parse(dr[i]["岗位"].ToString());

                using (StudyPlatformEntities db = new StudyPlatformEntities())
                {
                    S_UserInfo model = new S_UserInfo();
                    model.brand = brand;
                    model.username = username;
                    model.mobile = mobile;
                    model.depotname = province + city + depot;
                    model.usertype = usertype;
                    model.crdate = DateTime.Now;
                    model.status = 1;
                    model.isreg = 0;
                    db.S_UserInfo.Add(model);
                    db.SaveChanges();
                    if (model.id > 0)
                        successly++;
                }
            }
            catch (Exception ex)
            {
                _Result = _Result + ex.InnerException + "\\n\\r";
            }
        }
        if (successly == rowsnum)
        {
            result = "Excle表导入成功!";
        }
        else
        {
            result = "导入失败";
        }
    }
    else
    {
        result = "Excel表数据为空";
    }
    return result;
}
```

### 2.上传图片

```
#region 图片方向修正
public static void RotateImage(Image img)
{
    PropertyItem[] exif = img.PropertyItems;
    byte orientation = 0;
    foreach (PropertyItem i in exif)
    {
        if (i.Id == 274)
        {
            orientation = i.Value[0];
            i.Value[0] = 1;
            img.SetPropertyItem(i);
        }
    }

    switch (orientation)
    {
        case 2:
            img.RotateFlip(RotateFlipType.RotateNoneFlipX);
            break;
        case 3:
            img.RotateFlip(RotateFlipType.Rotate180FlipNone);
            break;
        case 4:
            img.RotateFlip(RotateFlipType.RotateNoneFlipY);
            break;
        case 5:
            img.RotateFlip(RotateFlipType.Rotate90FlipX);
            break;
        case 6:
            img.RotateFlip(RotateFlipType.Rotate90FlipNone);
            break;
        case 7:
            img.RotateFlip(RotateFlipType.Rotate270FlipX);
            break;
        case 8:
            img.RotateFlip(RotateFlipType.Rotate270FlipNone);
            break;
        default:
            break;
    }
    foreach (PropertyItem i in exif)
    {
        if (i.Id == 40962)
        {
            i.Value = BitConverter.GetBytes(img.Width);
        }
        else if (i.Id == 40963)
        {
            i.Value = BitConverter.GetBytes(img.Height);
        }
    }
}
#endregion

#region 图片压缩(直接)
/// 图片压缩(直接)  
/// <param name="img">原图片</param>  
/// <param name="dFile">压缩后保存位置</param>  
/// <param name="dHeight">高度</param>  
/// <param name="dWidth"></param>  
/// <param name="flag">压缩质量(数字越小压缩率越高) 1-100</param>  
/// <returns></returns>  
public static bool GetPicCp(Image img, string dFile, int dHeight, int dWidth, int flag)
{
    PropertyItem[] pt = img.PropertyItems;
    ImageFormat tFormat = img.RawFormat;
    int sW = 0, sH = 0;

    //按比例缩放
    Size tem_size = new Size(img.Width, img.Height);

    if (tem_size.Width > dHeight || tem_size.Width > dWidth)
    {
        if ((tem_size.Width * dHeight) > (tem_size.Width * dWidth))
        {
            sW = dWidth;
            sH = (dWidth * tem_size.Height) / tem_size.Width;
        }
        else
        {
            sH = dHeight;
            sW = (tem_size.Width * dHeight) / tem_size.Height;
        }
    }
    else
    {
        sW = tem_size.Width;
        sH = tem_size.Height;
    }

    Bitmap ob = new Bitmap(sW, sH);
    foreach (PropertyItem p in pt)
    {
        ob.SetPropertyItem(p);
    }
    Graphics g = Graphics.FromImage(ob);

    g.Clear(Color.WhiteSmoke);
    g.CompositingQuality = CompositingQuality.HighQuality;
    g.SmoothingMode = SmoothingMode.HighQuality;
    g.InterpolationMode = InterpolationMode.HighQualityBicubic;

    g.DrawImage(img, new Rectangle(0, 0, sW, sH), 0, 0, img.Width, img.Height, GraphicsUnit.Pixel);

    g.Dispose();
    //以下代码为保存图片时，设置压缩质量  
    EncoderParameters ep = new EncoderParameters();
    long[] qy = new long[1];
    qy[0] = flag;//设置压缩的比例1-100  
    EncoderParameter eParam = new EncoderParameter(System.Drawing.Imaging.Encoder.Quality, qy);
    ep.Param[0] = eParam;
    try
    {
        ImageCodecInfo[] arrayICI = ImageCodecInfo.GetImageEncoders();
        ImageCodecInfo jpegICIinfo = null;
        for (int x = 0; x < arrayICI.Length; x++)
        {
            if (arrayICI[x].FormatDescription.Equals("JPEG"))
            {
                jpegICIinfo = arrayICI[x];
                break;
            }
        }
        if (jpegICIinfo != null)
        {
            ob.Save(dFile, jpegICIinfo, ep);//dFile是压缩后的新路径  
        }
        else
        {
            ob.Save(dFile, tFormat);
        }
        return true;
    }
    catch
    {
        return false;
    }
    finally
    {
        //img.Dispose();
        ob.Dispose();
    }
}
#endregion

#region 上传图片(活动封面长方形)
/// <summary>
/// 上传图片(活动封面长方形)
/// </summary>
/// <returns></returns>
public static string uploadimgcpcover(HttpRequest request)
{
    string result = string.Empty;
    string dir = string.Empty;
    string uploadPath = string.Empty;
    string filename = string.Empty;
    string suffix = string.Empty;
    string fullname = string.Empty;
    //HttpRequest request = System.Web.HttpContext.Current.Request;
    HttpFileCollection fileCollection = request.Files;
    string brand = request.Form[0];
    string category = request.Form[1];
    //foreach(string key in request.Form.AllKeys)
    //{

    //}
    dir = "/upload/" + brand + "/"+ category + "/" + DateTime.Today.ToString("yyyyMMdd") + "/";
    uploadPath = AppDomain.CurrentDomain.SetupInformation.ApplicationBase + dir;
    if (fileCollection.Count == 1)
    {
        HttpPostedFile httpPostedFile = fileCollection[0];
        int filesize = httpPostedFile.ContentLength;
        if (filesize < 20971520)
        {
            try
            {
                using (Image image = new Bitmap(httpPostedFile.InputStream))
                {
                    RotateImage(image);
                    filename = Guid.NewGuid().ToString();
                    suffix = Path.GetExtension(httpPostedFile.FileName);
                    if (suffix == "")
                    {
                        suffix = ".jpg";
                    }
                    if (!Directory.Exists(uploadPath))
                    {
                        Directory.CreateDirectory(uploadPath);
                    }
                    fullname = filename + suffix;
                    string filePath = uploadPath + fullname;
                    while (File.Exists(filePath))
                    {
                        filename = Guid.NewGuid().ToString();
                        fullname = filename + suffix;
                        filePath = uploadPath + fullname;
                    }
                    bool bl = GetPicCp(image, filePath, 900, 900, 60);
                    if (bl == true)
                    {
                        result =  dir + fullname;
                    }
                    else
                    {
                        image.Save(filePath);
                        result =  dir + fullname;
                    }
                }
            }
            catch (Exception)
            {
                result = "不是图片格式！";
            }

        }
        else
        {
            result = "文件大小超限！";
        }
    }
    else
    {
        result = "文件数量不对！";
    }
    return result;
}
#endregion
```

### 3.防sql注入

```
#region 防sql注入过滤
/// <summary>
/// sql关键字过滤
/// </summary>
/// <param name="InText"></param>
/// <returns></returns>
public static bool SqlFilter(string InText)
{
    string word = "and|or|exec|sp_executesql|insert|select|delete|drop|update|chr|mid|master|truncate|char|declare|join|cmd|;|'|%|=|<|>|--|xp_cmdshell";
    if (InText == null)
        return false;
    foreach (string i in word.Split('|'))
    {
        if ((InText.ToLower().IndexOf(i + " ") > -1) || (InText.ToLower().IndexOf(" " + i) > -1))
        {
            return true;
        }
    }
    return false;
}
#endregion
```

### 4.Json字符串转换为 DataTable数据集合

```
#region Json字符串转换为 DataTable数据集合
/// <summary>
/// Json 字符串 转换为 DataTable数据集合
/// </summary>
/// <param name="json"></param>
/// <returns></returns>
public DataTable ToDataTable(string json)
{
    DataTable dataTable = new DataTable();  //实例化
    DataTable result;
    try
    {
        JavaScriptSerializer javaScriptSerializer = new JavaScriptSerializer();
        javaScriptSerializer.MaxJsonLength = Int32.MaxValue; //取得最大数值
        ArrayList arrayList = javaScriptSerializer.Deserialize<ArrayList>(json);
        if (arrayList.Count > 0)
        {
            foreach (Dictionary<string, object> dictionary in arrayList)
            {
                if (dictionary.Keys.Count<string>() == 0)
                {
                    result = dataTable;
                    return result;
                }
                //Columns
                if (dataTable.Columns.Count == 0)
                {
                    foreach (string current in dictionary.Keys)
                    {
                        if (current != "data")
                            dataTable.Columns.Add(current, dictionary[current].GetType());
                        else
                        {
                            ArrayList list = dictionary[current] as ArrayList;
                            foreach (Dictionary<string, object> dic in list)
                            {
                                foreach (string key in dic.Keys)
                                {
                                    dataTable.Columns.Add(key, dic[key].GetType());
                                }
                                break;
                            }
                        }
                    }
                }
                //Rows
                ArrayList aList = new ArrayList();
                foreach (string current in dictionary.Keys)
                {
                    if (current != "data")
                    {
                        aList.Add(dictionary[current].ToString());
                    }
                    else
                    {
                        ArrayList list = dictionary[current] as ArrayList;
                        foreach (Dictionary<string, object> dic in list)
                        {
                            DataRow dataRow = dataTable.NewRow();

                            for (int i = 0; i < aList.Count; i++)
                            {
                                dataRow[i] = aList[i];
                            }
                            foreach (string key in dic.Keys)
                            {
                                dataRow[key] = dic[key];
                            }
                            dataTable.Rows.Add(dataRow);
                        }
                    }
                }
            }
        }
    }
    catch
    {
    }
    result = dataTable;
    return result;
}
#endregion
```

### 5.两层Json字符串转换为 DataTable数据集合

```
#region 两层Json字符串转换为 DataTable数据集合
/// <summary>
/// 两层Json字符串转换为 DataTable数据集合
/// </summary>
/// <param name="jsonstr"></param>
/// <returns></returns>
public DataTable JsonToDataTable2(string jsonstr)
{
    DataTable dataTable = new DataTable();  //实例化
    DataTable result;
    try
    {
        JavaScriptSerializer javaScriptSerializer = new JavaScriptSerializer();
        javaScriptSerializer.MaxJsonLength = Int32.MaxValue; //取得最大数值
        ArrayList arrayList = javaScriptSerializer.Deserialize<ArrayList>(jsonstr);
        if (arrayList.Count > 0)
        {
            foreach (Dictionary<string, object> dictionary in arrayList)
            {
                if (dictionary.Keys.Count<string>() == 0)
                {
                    result = dataTable;
                    return result;
                }
                //Columns
                if (dataTable.Columns.Count == 0)
                {
                    foreach (string current in dictionary.Keys)
                    {
                        if (current != "data")
                            dataTable.Columns.Add(current, dictionary[current].GetType());
                        else
                        {
                            ArrayList list = dictionary[current] as ArrayList;
                            foreach (Dictionary<string, object> dic in list)
                            {
                                foreach (string key in dic.Keys)
                                {
                                    dataTable.Columns.Add(key, dic[key].GetType());
                                }
                                break;
                            }
                        }
                    }
                }
                //Rows
                ArrayList aList = new ArrayList();
                foreach (string current in dictionary.Keys)
                {
                    if (current != "data")
                    {
                        aList.Add(dictionary[current].ToString());
                    }
                    else
                    {
                        ArrayList list = dictionary[current] as ArrayList;
                        foreach (Dictionary<string, object> dic in list)
                        {
                            DataRow dataRow = dataTable.NewRow();
                            for (int i = 0; i < aList.Count; i++)
                            {
                                dataRow[i] = aList[i];
                            }
                            foreach (string key in dic.Keys)
                            {
                                dataRow[key] = dic[key];
                            }
                            dataTable.Rows.Add(dataRow);
                        }
                    }
                }
            }
        }
    }
    catch
    {
    }
    result = dataTable;
    return result;
}
#endregion
```

### 6.获取随机数

```
#region 获取随机字符串
/// <summary>
///获取随机字符串
/// </summary>
public static string getStr(bool b, int n)//b：是否有复杂字符，n：生成的字符串长度

{

    string str = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    if (b == true)
    {
        str += "!\"#$%&'()*+,-./:;<=>?@[\\]^_`{|}~";//复杂字符
    }
    StringBuilder SB = new StringBuilder();
    Random rd = new Random();
    for (int i = 0; i < n; i++)
    {
        SB.Append(str.Substring(rd.Next(0, str.Length), 1));
    }
    return SB.ToString();

}
#endregion
```

### 7.获取二维码

```
#region 获取二维码
/// <summary>
/// 获取二维码
/// </summary>
/// <returns></returns>
public HttpResponseMessage GetQrCode( string url,string id,int size)
{
    if (url != "" &&url!=null && id != "" &&id!=null)
    {
        string qrcodeurl = url + "?id=" + id;
        System.Drawing.Image image = CreateQRCode(qrcodeurl,
         QRCodeEncoder.ENCODE_MODE.BYTE,
         QRCodeEncoder.ERROR_CORRECTION.M,
         8,
         6,
         size,
         5, Color.Green);
        System.IO.MemoryStream ms = new System.IO.MemoryStream();
        image.Save(ms, System.Drawing.Imaging.ImageFormat.Png);

        byte[] img = new byte[ms.Length];
        ms.Position = 0;
        ms.Read(img, 0, Convert.ToInt32(ms.Length));
        ms.Close();
        var resp = new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new ByteArrayContent(img)
        };
        resp.Content.Headers.ContentType = new MediaTypeHeaderValue("image/jpg");
        return resp;
    }
    else
    {
        var resp = new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new ByteArrayContent(System.Text.Encoding.Default.GetBytes("参数不完整"))
        };
        resp.Content.Headers.ContentType = new MediaTypeHeaderValue("text/plain");
        return resp;
    }
}

/// <summary>
/// CreateQRCode
/// </summary>
/// <param name="Content"></param>
/// <param name="QRCodeEncodeMode"></param>
/// <param name="QRCodeErrorCorrect"></param>
/// <param name="QRCodeVersion"></param>
/// <param name="QRCodeScale"></param>
/// <param name="size"></param>
/// <param name="border"></param>
/// <param name="codeEyeColor"></param>
/// <returns></returns>
private System.Drawing.Image CreateQRCode(string Content, QRCodeEncoder.ENCODE_MODE QRCodeEncodeMode, QRCodeEncoder.ERROR_CORRECTION QRCodeErrorCorrect, int QRCodeVersion, int QRCodeScale, int size, int border, Color codeEyeColor)
{
    QRCodeEncoder qrCodeEncoder = new QRCodeEncoder();
    qrCodeEncoder.QRCodeEncodeMode = QRCodeEncodeMode;
    qrCodeEncoder.QRCodeErrorCorrect = QRCodeErrorCorrect;
    qrCodeEncoder.QRCodeScale = QRCodeScale;
    qrCodeEncoder.QRCodeVersion = QRCodeVersion;

    System.Drawing.Image image = qrCodeEncoder.Encode(Content);

    #region 根据设定的目标图片尺寸调整二维码QRCodeScale设置，并添加边框
    if (size > 0)
    {
        //当设定目标图片尺寸大于生成的尺寸时，逐步增大方格尺寸
        #region 当设定目标图片尺寸大于生成的尺寸时，逐步增大方格尺寸
        while (image.Width < size)
        {
            qrCodeEncoder.QRCodeScale++;
            System.Drawing.Image imageNew = qrCodeEncoder.Encode(Content);
            if (imageNew.Width < size)
            {
                image = new System.Drawing.Bitmap(imageNew);
                imageNew.Dispose();
                imageNew = null;
            }
            else
            {
                qrCodeEncoder.QRCodeScale--; //新尺寸未采用，恢复最终使用的尺寸
                imageNew.Dispose();
                imageNew = null;
                break;
            }
        }
        #endregion

        //当设定目标图片尺寸小于生成的尺寸时，逐步减小方格尺寸
        #region 当设定目标图片尺寸小于生成的尺寸时，逐步减小方格尺寸
        while (image.Width > size && qrCodeEncoder.QRCodeScale > 1)
        {
            qrCodeEncoder.QRCodeScale--;
            System.Drawing.Image imageNew = qrCodeEncoder.Encode(Content);
            image = new System.Drawing.Bitmap(imageNew);
            imageNew.Dispose();
            imageNew = null;
            if (image.Width < size)
            {
                break;
            }
        }
        #endregion

        //根据参数设置二维码图片白边的最小宽度（按需缩小）
        #region 根据参数设置二维码图片白边的最小宽度
        if (image.Width <= size && border > 0)
        {
            while (image.Width <= size && size - image.Width < border * 2 && qrCodeEncoder.QRCodeScale > 1)
            {
                qrCodeEncoder.QRCodeScale--;
                System.Drawing.Image imageNew = qrCodeEncoder.Encode(Content);
                image = new System.Drawing.Bitmap(imageNew);
                imageNew.Dispose();
                imageNew = null;
            }
        }
        #endregion

        //已经确认二维码图像，为图像染色修饰
        if (true)
        {
            //定位点方块边长
            int beSize = qrCodeEncoder.QRCodeScale * 3;

            int bep1_l = qrCodeEncoder.QRCodeScale * 2;
            int bep1_t = qrCodeEncoder.QRCodeScale * 2;

            int bep2_l = image.Width - qrCodeEncoder.QRCodeScale * 5 - 1;
            int bep2_t = qrCodeEncoder.QRCodeScale * 2;

            int bep3_l = qrCodeEncoder.QRCodeScale * 2;
            int bep3_t = image.Height - qrCodeEncoder.QRCodeScale * 5 - 1;

            int bep4_l = image.Width - qrCodeEncoder.QRCodeScale * 7 - 1;
            int bep4_t = image.Height - qrCodeEncoder.QRCodeScale * 7 - 1;

            System.Drawing.Graphics graphic0 = System.Drawing.Graphics.FromImage(image);

            // Create solid brush. 
            SolidBrush blueBrush = new SolidBrush(codeEyeColor);

            // Fill rectangle to screen. 
            graphic0.FillRectangle(blueBrush, bep1_l, bep1_t, beSize, beSize);
            graphic0.FillRectangle(blueBrush, bep2_l, bep2_t, beSize, beSize);
            graphic0.FillRectangle(blueBrush, bep3_l, bep3_t, beSize, beSize);
            graphic0.FillRectangle(blueBrush, bep4_l, bep4_t, qrCodeEncoder.QRCodeScale, qrCodeEncoder.QRCodeScale);
        }

        //当目标图片尺寸大于二维码尺寸时，将二维码绘制在目标尺寸白色画布的中心位置
        #region 如果目标尺寸大于生成的图片尺寸，将二维码绘制在目标尺寸白色画布的中心位置
        if (image.Width < size)
        {
            //新建空白绘图
            System.Drawing.Bitmap panel = new System.Drawing.Bitmap(size, size);
            System.Drawing.Graphics graphic0 = System.Drawing.Graphics.FromImage(panel);
            int p_left = 0;
            int p_top = 0;
            if (image.Width <= size) //如果原图比目标形状宽
            {
                p_left = (size - image.Width) / 2;
            }
            if (image.Height <= size)
            {
                p_top = (size - image.Height) / 2;
            }

            //将生成的二维码图像粘贴至绘图的中心位置
            graphic0.DrawImage(image, p_left, p_top, image.Width, image.Height);
            image = new System.Drawing.Bitmap(panel);
            panel.Dispose();
            panel = null;
            graphic0.Dispose();
            graphic0 = null;
        }
        #endregion
    }
    #endregion
    return image;

//var imgPath = @"D:\ITdosCom\Images\itdos.jpg";
////从图片中读取byte  
//var imgByte = File.ReadAllBytes(imgPath);
////从图片中读取流  
//var imgStream = new MemoryStream(File.ReadAllBytes(imgPath));
//var resp = new HttpResponseMessage(HttpStatusCode.OK)
//{
//    Content = new ByteArrayContent(imgByte)
//    //或者  
//    //Content = new StreamContent(stream)  
//};
//resp.Content.Headers.ContentType = new MediaTypeHeaderValue("image/jpg");
//return resp;
} 
#endregion
```
