## 1. Socket kommunik�ci� kliens �s szerver k�z�tt

A szervern�l kisebb az es�lye, hogy NAT-olva van, ez�rt c�lszer� azt befogni szervernek, �s a kliensr�l hozz� csatlakozni az interneten kereszt�l, �s byte streamet beolvasni/k�ldeni.

���server = new TcpListener(IPAddress.Any, port);
server.Start();
while (client == null || client.Connected == false)
{
	client=server.AcceptTcpClient();
}
stream = client.GetStream();
while ((i = stream.Read(bytes, 0, bytes.Length)) != 0)
{
	string[] dataArray=data.Split(new[] { "<sprtr>" }, StringSplitOptions.None);
	//initial setup needed when connecting
	if (dataArray[0]=="setup")
        {
        	setupInit();
        }
	//the user selected another app on the client
        else if (dataArray[0]=="selected")
        {
        	if (SpinnerSelectedEvent != null)
        		SpinnerSelectedEvent(dataArray[1]); //send the app specific shortcuts
        }
        //received a keystroke which should be processed and executed
        else if (dataArray[0]=="keystroke")
        {
        	wrapper.Send(dataArray[1]);
        }
}���

## 2. Billenty�lenyom�sok szimul�l�sa k�db�l

Windows alatt van erre egy rendk�v�l j�l meg�rt API, a [SendKeys](https://msdn.microsoft.com/en-us/library/system.windows.forms.sendkeys(v=vs.110).aspx). Erre �rtam egy wrappert, ami a gyakran haszn�lt kifejez�seket leegyszer�s�ti. A �(seq)(/seq)� tagek k�z�tt l�v� sz�veget alak�tja �t, �s k�ldi ki egyszerre, billenty�kombin�ci�k�nt is. Pl: �(seq)\s\f2(/seq)� a Shift+F2 kombin�ci�t eredm�nyezi (�+{F2}� SendKeys szintaxis szerint). Ha nem F billenty� �ll a v�g�n, akkor m�s karakter is �llhat ut�na, pl �\c\sf� Ctrl+Shift+F lesz, de haszn�lhat� a SendKeys szintaxis is a seq tagen bel�l.

Seq tagen bel�li szintaxis kezel�f�ggv�ny r�szlete:
�//key sequences must be sent together
private void SendFast(string toSend)
{
	StringBuilder output = new StringBuilder();

	//split before each escaped character
	string[] array = toSend.Split(new[] { "\\" }, StringSplitOptions.None);
	foreach (string s in array)
	{
		//if it's an empty string, there's no need to check
		//E.g. "\\n".Split(..."\\"...); will have an empty string as 0th member because
                //the first occurrence is at the beginning of the string
                if (s.Length != 0)
                    switch (s[0])
                    {
                        //convert to SendKeys semantics and append if it has more chars
                        case 's': //shift
                            output.Append("+"); output.Append(s.Substring(1)); break;
                        case 'c': //ctrl
                            output.Append("^"); output.Append(s.Substring(1)); break;
                        case 'a': //alt
                            output.Append("%"); output.Append(s.Substring(1)); break;
...
	SendKeys.SendWait(output.ToString());
}�

A seq tagen k�v�l l�v� karaktereket k�l�n szimul�lja a szerver, �s k�z�tt�k random id�k�zt v�r, hogy �gy t�nj�n, mintha egy val�di ember g�peln�. Ha csak billenty�kombin�ci�t k�ld�nk, akkor seq tagen bel�l adunk inform�ci�t, �gy ez a billenty�kombin�ci�kat nem �rinti.

�//wait between each character, simulating a real person typing
private void SendSlow(string s)
{
	SendKeys.SendWait(s);
	Thread.Sleep(rand.Next(25, 100));
}�

N�h�ny karakternek a SendKeys szintaxison bel�l saj�tos jelent�se van, �gy azokat escapelni kell {} tagek k�z�. Ilyen a {, }, (, ), �s m�g m�s kev�sb� haszn�ltak. Ezek nyilv�nval�an a lass� sz�gvegn�l jelentenek probl�m�t, mivel billenty�kombin�ci�ban nem gyakran szerepel "{" **billenty�**.

�switch (c)
{
	//cases for single characters that need special wrapping in SendKeys
	case '{':
	{
		//these must be sent together
		SendFast("{{}");
	}
	break;
	case '}':
	{
		SendFast("{}}");
	}
	break;�