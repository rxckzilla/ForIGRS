
/*
 * $Id: EchoServlet.java,v 1.5 2003/06/22 12:32:15 fukuda Exp $
 */
package org.mobicents.servlet.sip.example;

import java.util.*;
import java.io.IOException;

import javax.servlet.sip.SipServlet;	
import javax.servlet.sip.SipServletRequest;
import javax.servlet.sip.SipServletResponse;
import javax.servlet.sip.SipSession;
import javax.servlet.sip.SipApplicationSession;
import javax.servlet.ServletException;
import javax.servlet.sip.URI;
import javax.servlet.sip.Proxy;
import javax.servlet.sip.SipFactory;
import java.nio.charset.StandardCharsets;

/**
 */
public class Myapp extends SipServlet {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	static private Map<String, String> EstadosDB;
	static private Map<String, String> RegistrarDB;
	static private SipFactory factory;
	
	public Myapp() {
		super();
		RegistrarDB = new HashMap<String,String>();
		EstadosDB = new HashMap<String, String>();
	}
	
	public void init() {
		factory = (SipFactory) getServletContext().getAttribute(SIP_FACTORY);
	}

	/**
        * Acts as a registrar and location service for REGISTER messages
        * @param  request The SIP message received by the AS 
        */
	protected void doRegister(SipServletRequest request) throws ServletException,
        IOException {
		String aor = getSIPuri(request.getHeader("To"));
		String contact = getSIPuriPort(request.getHeader("Contact"));

		// Perform authentication and authorization checks here.
		// You may need to check credentials or other security mechanisms.

		// For example:
		// if (!isAuthorized(aor, request)) {
		//    SipServletResponse response = request.createResponse(403);
		//    response.send();
		//    return;
		// }

		if (!aor.endsWith("@acme.pt")) {
			log("Invalid domain in the SIP request: " + aor);
			SipServletResponse response = request.createResponse(403);
			response.send();
			return;
		}
		if ("0".equals(request.getHeader("Expires"))) {
			// Deregistration request, remove the AOR from the RegistrarDB
			RegistrarDB.remove(aor);
			log("Deregistered AOR: " + aor);
		} else {
			// Registration request, add the AOR to the RegistrarDB
			RegistrarDB.put(aor, contact);
			log("Registered AOR: " + aor);
		}
		
		SipServletResponse response = request.createResponse(200);
		response.send();

		// Some logs to show the content of the Registrar database.
		log("REGISTER (myapp):***");
		Iterator<Map.Entry<String, String>> it = RegistrarDB.entrySet().iterator();
		while (it.hasNext()) {
			Map.Entry<String, String> pairs = it.next();
			System.out.println(pairs.getKey() + " = " + pairs.getValue());
		}
		log("REGISTER (myapp):***");
	}


	/**
        * Sends SIP replies to INVITE messages
        * - 300 if registred
        * - 404 if not registred
        * @param  request The SIP message received by the AS 
        */
	protected void doInvite(SipServletRequest request)
                  throws ServletException, IOException {
		
		// Some logs to show the content of the Registrar database.
		log("INVITE (myapp):***");
		Iterator<Map.Entry<String,String>> it = RegistrarDB.entrySet().iterator();
    		while (it.hasNext()) {
        		Map.Entry<String,String> pairs = (Map.Entry<String,String>)it.next();
        		System.out.println(pairs.getKey() + " = " + pairs.getValue());
    		}
		log("INVITE (myapp):***");
		
		/*
		String aor = getSIPuri(request.getHeader("To")); // Get the To AoR
	    if (!RegistrarDB.containsKey(aor)) { // To AoR not in the database, reply 404
			SipServletResponse response; 
			response = request.createResponse(404);
			response.send();
	    } else {
			SipServletResponse response = request.createResponse(300);
			// Get the To AoR contact from the database and add it to the response 
			response.setHeader("Contact",RegistrarDB.get(aor));
			response.send();
		}
		SipServletResponse response = request.createResponse(404);
		response.send();
		*/
		String aor = getSIPuri(request.getHeader("To")); // Get the To AoR
		String domain = aor.substring(aor.indexOf("@")+1, aor.length());
		log(domain);
		if (domain.equals("acme.pt")) { // The To domain is the same as the server 
	    	if (!RegistrarDB.containsKey(aor)) { // To AoR not in the database, reply 404
				SipServletResponse response; 
				response = request.createResponse(404);
				response.send();
	    	} else {
				Proxy proxy = request.getProxy();
				proxy.setRecordRoute(false);
				proxy.setSupervised(false);
				URI toContact = factory.createURI(RegistrarDB.get(aor));
				proxy.proxyTo(toContact);
			}			
		} else {
			Proxy proxy = request.getProxy();
			proxy.proxyTo(request.getRequestURI());
		}

		/*
	    if (!RegistrarDB.containsKey(aor)) { // To AoR not in the database, reply 404
			SipServletResponse response; 
			response = request.createResponse(404);
			response.send();
	    } else {
			SipServletResponse response = request.createResponse(300);
			// Get the To AoR contact from the database and add it to the response 
			response.setHeader("Contact",RegistrarDB.get(aor));
			response.send();
		}
		*/
	}

	/**
	* Acts as a registrar and location service for REGISTER messages
	* @param  request The SIP message received by the AS 
	*/
	
	 protected void doMessage(SipServletRequest request) throws ServletException, IOException {
		String aorFrom = getSIPuri(request.getHeader("From"));
		String aorTo = getSIPuri(request.getHeader("To"));

		// Log the AORs for debugging
		log("AOR From: " + aorFrom);
		log("AOR To: " + aorTo);

		if (!aorFrom.endsWith("@acme.pt") || !aorTo.endsWith("@acme.pt")) {
			// User does not belong to the restricted group, indicate that the service is not available
			SipServletResponse response = request.createResponse(403);
			response.send();
			return;
		}
		String targetContact = (String) request.getContent();
		if (RegistrarDB.containsKey(targetContact)) {
			// AOR To exists, create a message session and respond with 200 OK

			if(aorTo.equals("sip:gofind@acme.pt")){
				SipSession sessionFrom = request.getSession();
				SipApplicationSession appSession = sessionFrom.getApplicationSession();

				// Retrieve the session for the recipient
				SipServletRequest messageRequest = factory.createRequest(
						appSession,
						"INVITE",
						aorFrom,
						targetContact
						//RegistrarDB.get(aorTo)
				);

				messageRequest.send();

				// Respond to the original MESSAGE request with 200 OK
				SipServletResponse response = request.createResponse(200);
				response.send();
			}else{
				SipServletResponse response = request.createResponse(403);
				response.send();
			}

			
		} else {
			// Log the RegistrarDB content for debugging
			log("RegistrarDB Content: " + RegistrarDB);

			// AOR To does not exist, respond with 404 Not Found
			SipServletResponse response = request.createResponse(404);
			response.send();
		}
	}

	/**
	 * Sends an alert message to the specified recipient.
	 *
	 * @param request       The SIP message received by the AS
	 * @param aorTo  The AOR of the recipient
	 * @param alertMessage  The alert message content
	 */
	private void sendAlertToRecipient(SipServletRequest request, String aorTo, String alertMessage)
            throws IOException, ServletException {
        if (RegistrarDB.containsKey(aorTo)) {
            SipServletRequest alertRequest = factory.createRequest(
                    request.getApplicationSession(),
                    "MESSAGE",
                    aorTo,
                    RegistrarDB.get(aorTo)
            );

            alertRequest.setContent(alertMessage.getBytes(StandardCharsets.UTF_8), "text/plain");
            alertRequest.send();
        }
    }

    // Other helper methods...

    /**
     * Extracts the desired recipient from the content of the MESSAGE request.
     * Implement this method based on the format/content of your messages.
     *
     * @param messageContent The content of the MESSAGE request
     * @return The extracted recipient AOR
     */
    private String extractRecipientFromMessage(String messageContent) {
        // Implement extraction logic based on your message format
        // For example, if your message is in the format "TO: user@acme.pt", you can extract "user@acme.pt".
        // Modify this method based on the actual message format you are using.
        return messageContent.trim();
    }

    /**
     * Gets the status of the selected pair.
     * Implement this method based on your business logic.
     *
     * @param aorTo The AOR of the selected recipient
     * @return The status of the selected pair
     */
    private String getStatusForRecipient(String aorTo) {
        // Implement your logic to determine the status (e.g., no answer, busy on call, in conference)
        // Modify this method based on your actual business logic.
        // For example, you can use information from your application state or database.
        return "No answer";
    } 
	



/*
        * Auxiliary function for extracting SPI URIs
        * @param  uri A URI with optional extra attributes 
        * @return SIP URI 
        */
	protected String getSIPuri(String uri) {
		String f = uri.substring(uri.indexOf("<")+1, uri.indexOf(">"));
		int indexCollon = f.indexOf(":", f.indexOf("@"));
		if (indexCollon != -1) {
			f = f.substring(0,indexCollon);
		}
		return f;
	}

	/**
        * Auxiliary function for extracting SPI URIs
        * @param  uri A URI with optional extra attributes 
        * @return SIP URI and port 
        */
	protected String getSIPuriPort(String uri) {
		String f = uri.substring(uri.indexOf("<")+1, uri.indexOf(">"));
		return f;
	}
}
