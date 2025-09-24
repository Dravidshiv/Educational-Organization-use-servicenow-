import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.Base64;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;

public class ServiceNowClient {
    private final HttpClient http;
    private final String instanceUrl; // e.g. https://dev12345.service-now.com
    private final String authHeader;  // either "Basic ..." or "Bearer <token>"
    private final ObjectMapper mapper = new ObjectMapper();

    public ServiceNowClient(String instanceUrl, String authHeader) {
        this.instanceUrl = instanceUrl;
        this.authHeader = authHeader;
        this.http = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    // Helper to post JSON to a ServiceNow table API
    private HttpResponse<String> postToTable(String tableName, String jsonBody) throws Exception {
        String url = instanceUrl + "/api/now/table/" + tableName;
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .timeout(Duration.ofSeconds(10))
                .header("Accept", "application/json")
                .header("Content-Type", "application/json")
                .header("Authorization", authHeader)
                .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
                .build();
        return http.send(request, HttpResponse.BodyHandlers.ofString());
    }

    // Create an "incident" as a student help ticket
    public String createStudentTicket(String shortDesc, String description, String studentId, String callerSysIdOrEmail) throws Exception {
        ObjectNode payload = mapper.createObjectNode();
        payload.put("short_description", shortDesc);
        payload.put("description", description);
        // map student-specific fields — replace with your custom field names if configured
        payload.put("u_student_id", studentId);          // example custom field
        payload.put("category", "student_services");     // example category
        payload.put("caller_id", callerSysIdOrEmail);    // caller sys_id or email depending on instance config

        HttpResponse<String> response = postToTable("incident", mapper.writeValueAsString(payload));
        if (response.statusCode() >= 200 && response.statusCode() < 300) {
            // return response body (ServiceNow returns created record details)
            return response.body();
        } else {
            throw new RuntimeException("Failed to create ticket. HTTP " + response.statusCode() + ": " + response.body());
        }
    }

    // Get an incident by sys_id
    public String getTicketBySysId(String sysId) throws Exception {
        String url = String.format("%s/api/now/table/incident/%s", instanceUrl, sysId);
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("Accept", "application/json")
                .header("Authorization", authHeader)
                .GET()
                .build();
        HttpResponse<String> response = http.send(request, HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() == 200) return response.body();
        throw new RuntimeException("Failed to fetch ticket: " + response.statusCode() + " " + response.body());
    }

    // Update an incident (fields map in JSON)
    public String updateTicket(String sysId, ObjectNode updateFields) throws Exception {
        String url = String.format("%s/api/now/table/incident/%s", instanceUrl, sysId);
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("Accept", "application/json")
                .header("Content-Type", "application/json")
                .header("Authorization", authHeader)
                .method("PATCH", HttpRequest.BodyPublishers.ofString(mapper.writeValueAsString(updateFields)))
                .build();
        HttpResponse<String> response = http.send(request, HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() >= 200 && response.statusCode() < 300) return response.body();
        throw new RuntimeException("Failed to update ticket: " + response.statusCode() + " " + response.body());
    }

    // Create a user (sys_user table) - useful to create student accounts or staff
    public String createUser(String firstName, String lastName, String email, String studentId) throws Exception {
        ObjectNode payload = mapper.createObjectNode();
        payload.put("first_name", firstName);
        payload.put("last_name", lastName);
        payload.put("email", email);
        // Example custom attribute
        payload.put("u_student_id", studentId);

        HttpResponse<String> response = postToTable("sys_user", mapper.writeValueAsString(payload));
        if (response.statusCode() >= 200 && response.statusCode() < 300) {
            return response.body();
        } else {
            throw new RuntimeException("Failed to create user: " + response.statusCode() + " " + response.body());
        }
    }

    // --- Example usage main method ---
    public static void main(String[]
