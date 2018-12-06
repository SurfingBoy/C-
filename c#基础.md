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
