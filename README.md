# book-manage-system
登陆界面，连接数据库
// LoginDlg.cpp : implementation file
//

#include "stdafx.h"
#include "BookManage.h"
#include "LoginDlg.h"

#ifdef _DEBUG
#define new DEBUG_NEW
#undef THIS_FILE
static char THIS_FILE[] = __FILE__;
#endif

/////////////////////////////////////////////////////////////////////////////
// CLoginDlg dialog


CLoginDlg::CLoginDlg(CWnd* pParent /*=NULL*/)
	: CDialog(CLoginDlg::IDD, pParent)
{
	//{{AFX_DATA_INIT(CLoginDlg)
	m_pwd = _T("");
	m_userid = _T("");
	//}}AFX_DATA_INIT
	m_pRecordset = NULL;
	m_pConnection = NULL;
	VariantInit(&varUserName);
}


void CLoginDlg::DoDataExchange(CDataExchange* pDX)
{
	CDialog::DoDataExchange(pDX);
	//{{AFX_DATA_MAP(CLoginDlg)
	DDX_Text(pDX, IDC_EDIT_PWD, m_pwd);
	DDX_Text(pDX, IDC_EDIT_USERID, m_userid);
	//}}AFX_DATA_MAP
	DDX_Control(pDX, IDC_BTN_LOGIN, m_btnLogin);
	DDX_Control(pDX, IDCANCEL, m_btnCancel);
}


BEGIN_MESSAGE_MAP(CLoginDlg, CDialog)
	//{{AFX_MSG_MAP(CLoginDlg)
	ON_BN_CLICKED(IDC_BTN_LOGIN, OnBtnLogin)
	ON_WM_CTLCOLOR()
	//}}AFX_MSG_MAP
END_MESSAGE_MAP()

/////////////////////////////////////////////////////////////////////////////
// CLoginDlg message handlers

//功能：登录时身份验证
//时间：2009-9-23
void CLoginDlg::OnBtnLogin() 
{
	// TODO: Add your control notification handler code here
	//HRESULT hr;
	UpdateData(TRUE);
	//m_pwd.TrimRight();     //把密码右边的空格去掉

	//char buf[20];
	CString strSRC = "Provider = SQLOLEDB.1; Integrated Security = SSPI; Persist Security Info=False; Initial Catalog = BookManage; Data Source = .\\SQLEXPRESS";
	_bstr_t strConnect = strSRC;
	//_variant_t var;
	CString strSQL = "";
	//CString strReaderType;
	_variant_t varRet;      //存储Output的参数
//	int index;

	//检查用户ID和密码是否为空,且检测空格
	if(m_userid.IsEmpty())
	{
		MessageBox("请输入您的用户编号!","温馨提示",
			MB_OK | MB_ICONEXCLAMATION);
		m_pwd.Empty();
		UpdateData(FALSE);
		return;
	}
 	if(m_pwd.IsEmpty())
	{
		MessageBox("请输入您的密码!","温馨提示",
			MB_OK | MB_ICONEXCLAMATION);
		m_pwd.Empty();
		UpdateData(FALSE);
		return;
	}
 /*	else if(" " == m_userid.Left(1) || " " == m_userid.Right(1))  //用户编号的空格不用检测
	{
		MessageBox("用户编号中不正确!","温馨提示",
			MB_OK | MB_ICONEXCLAMATION);
		//m_userid.Empty();
		m_pwd.Empty();
		UpdateData(FALSE);
		return;
	}*/
	else if(" " == m_pwd.Left(1) || " " == m_pwd.Right(1))       //密码的空格需要检测
	{
		MessageBox("密码不正确!","温馨提示",
			MB_OK | MB_ICONEXCLAMATION);
		//m_userid.Empty();
		m_pwd.Empty();
		UpdateData(FALSE);
		return;
	}
		
	//连接数据库
	try 
	{ 
		m_pConnection.CreateInstance("ADODB.Connection");  //创建实例
		//身份验证模式为:"sql server和windows"
		//_bstr_t strConnect = "Provider = SQLOLEDB.1;Persist Security Info = True;User ID = sa;Password = 123;Initial Catalog = BookManage;Data Source=.";
		//身份验证模式为:"仅windows"
		m_pConnection->Open(strConnect, "", "", adModeUnknown); 
		//if( m_pConnection->State==adStateOpen)  
			//MessageBox("连接数据库"); 
		/*if( m_pConnection->State==adStateClosed)  
			MessageBox("断开连接"); */

		//创建存储过程的命令对象
		m_pCommand.CreateInstance("ADODB.Command");       //创建实例
		m_pCommand->ActiveConnection = m_pConnection;	  //设置连接
		m_pCommand->CommandText = "usp_Login";			  //存储过程为usp_Login

		//建立传入存储过程的参数
		//m_pParamID.CreateInstance("ADODB.Parameter");
		//m_pParamPwd.CreateInstance("ADODB.Parameter");
		//m_pParamRet.CreateInstance("ADODB.Parameter");
		
		m_pParamID = m_pCommand->CreateParameter("rdID",adInteger,adParamInput,-1,(_variant_t)m_userid);  //给参数设置属性
		m_pCommand->Parameters->Append(m_pParamID);		 //加入到Command对象的参数集属性中
	
		m_pParamPwd = m_pCommand->CreateParameter("rdPwd",adVarChar,adParamInput,10,(_variant_t)m_pwd);
		m_pCommand->Parameters->Append(m_pParamPwd);

 		m_pParamRet = m_pCommand->CreateParameter("ret",adChar,adParamReturnValue,1);
		m_pCommand->Parameters->Append(m_pParamRet);

		m_pCommand->Execute(NULL,NULL,adCmdStoredProc);

	    varRet = m_pCommand->Parameters->GetItem("ret")->GetValue();
		varRet.ChangeType(VT_I4, NULL);

		if(V_I4(&varRet))							   //通过身份验证
		{
			//MessageBox("登录成功！");
			//CString strType;

			//strType = m_userid.Left(1);                //获取用户ID首字母
			m_pRecordset.CreateInstance("ADODB.Recordset");

			                                           //判断是否是管理员登录
			strSQL.Format("select * from Manager where mgID = '%s'", m_userid);
			m_pRecordset->Open((_bstr_t)strSQL,m_pConnection.GetInterfacePtr(),adOpenStatic,adLockOptimistic,adCmdText);
			
			while(!m_pRecordset->adoEOF)
			{
				varUserType = m_pRecordset->GetCollect("mgType");        //获得管理员类型
				varUserName = m_pRecordset->GetCollect("mgName");		 //获得管理员名
				m_pRecordset->MoveNext();
			}

			if(m_pRecordset->State)
				m_pRecordset->Close();
			 
			if(NULL == varUserName.vt)								     //读者登录
			{
				strSQL.Format("select * from Reader where rdID = '%s'", m_userid);
				m_pRecordset->Open((_bstr_t)strSQL,m_pConnection.GetInterfacePtr(),adOpenStatic,adLockOptimistic,adCmdText);
				
				while(!m_pRecordset->adoEOF)
				{
					varUserType = m_pRecordset->GetCollect("rdType");    //获得读者类型
					varUserName = m_pRecordset->GetCollect("rdName");	 //获得读者名
					m_pRecordset->MoveNext();
				}
			}

		}
		else
		{
			MessageBox("用户编号或密码不正确！","温馨提示",
			MB_OK | MB_ICONEXCLAMATION);
			m_pwd = "";
			UpdateData(FALSE);

			return;
		}
	} 

	catch(_com_error e) 
	{ 
		AfxMessageBox(e.ErrorMessage()); 
		// 显示错误信息
		AfxMessageBox(e.Description());
		return;
	} 

	if(m_pRecordset->State)
		m_pRecordset->Close();

	this->EndDialog(TRUE);
}


BOOL CLoginDlg::OnInitDialog() 
{
	CDialog::OnInitDialog();
	
	// TODO: Add extra initialization here
	m_btnLogin.SetShade(CShadeButtonST::SHS_HSHADE,8,20,5,RGB(55,55,255));
	m_btnLogin.DrawFlatFocus(TRUE);

	m_btnCancel.SetShade(CShadeButtonST::SHS_HSHADE,8,20,5,RGB(55,55,255));
	m_btnCancel.DrawFlatFocus(TRUE);
	
	m_brush.CreateSolidBrush(RGB(208, 196, 174));
	return TRUE;  // return TRUE unless you set the focus to a control
	              // EXCEPTION: OCX Property Pages should return FALSE
}

HBRUSH CLoginDlg::OnCtlColor(CDC* pDC, CWnd* pWnd, UINT nCtlColor) 
{
	/*HBRUSH hbr = CDialog::OnCtlColor(pDC, pWnd, nCtlColor);
	
	// TODO: Change any attributes of the DC here
	
	// TODO: Return a different brush if the default is not desired
	return hbr;*/
	if(pWnd->GetDlgCtrlID() == IDC_STATIC1 ||  
		pWnd->GetDlgCtrlID() == IDC_STATIC2)
	{	
		pDC->SetBkColor(RGB(208, 196, 174));
	
	}

	return m_brush;   //返加绿色刷子
}
