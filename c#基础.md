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
