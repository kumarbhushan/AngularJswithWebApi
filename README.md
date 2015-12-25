Introduction
This sample shows how to create a Web API 2 controller that supports paging. It also shows two approaches to using the paging functionality with an AngularJS client.

simple paging - show a single list of n items to the user and if more items are available allow the user to get the next page of n items until all have been returned.
full paging - divide the data into pages of n items. The items for each page are loaded when the page is first viewed by the user.
Both approaches support changing the page size and sorting of the data.

Please note that all code snippets in this description are partial and only contain the relevant lines of code. To view the full code please download the sample. Also the approach taken in this sample has been to make the code as clear as possible to highlight the overall concept. In a real world scenario another design may preferable.

Building the Sample
This sample has the solution level NuGet packages removed to make it smaller for downloading. If you do not have NuGet Package Restore enabled run NuGet and it should prompt you to restore the missing packages.

Description
The sample consists of a single Web API project WebApiPagingAngularClient which contains the Web API controllers and the AgnularJS SPA. When running the project the default route ../Home/Index will load the sample spa. 

Server side (Web API)

Firstly there are couple of configuration changes to mention. In BundleConfig the line BundleTable.EnableOptimizations = true; has been commented out. This will stop the SPA scripts (bundled as app) from being bundled and minified, making it easier to step through the client code when it is running in browser. Also in WebApiConfig the JSON formatting has been set to camelcase.

C#
    public static class WebApiConfig 
    { 
        public static void Register(HttpConfiguration config) 
        {            
            // Use camelCase for JSON data. 
            config.Formatters.JsonFormatter.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver(); 
 
        } 
    }
 
The Web API has a controller ClubsController which provides actions to get a list of Clubs from a ClubRepository (a simple demo repository which just returns a List<Club> to provide some data for the demo). The GET action which supports paging is show below.
C#
// GET: api/Clubs/pageSize/pageNumber/orderBy(optional) 
[Route("{pageSize:int}/{pageNumber:int}/{orderBy:alpha?}")] 
public IHttpActionResult Get(int pageSize, int pageNumber, string orderBy = "") 
{ 
    var totalCount = this.clubRepository.Clubs.Count(); 
    var totalPages = Math.Ceiling((double)totalCount / pageSize); 
 
    var clubQuery = this.clubRepository.Clubs; 
 
    if (QueryHelper.PropertyExists<Club>(orderBy)) 
    { 
        var orderByExpression = QueryHelper.GetPropertyExpression<Club>(orderBy); 
        clubQuery = clubQuery.OrderBy(orderByExpression); 
    } else 
    { 
        clubQuery = clubQuery.OrderBy(c => c.Id); 
    } 
 
    var clubs = clubQuery.Skip((pageNumber - 1) * pageSize)                             
                            .Take(pageSize)                 
                            .ToList(); 
 
    var result = new 
    { 
        TotalCount = totalCount, 
        totalPages = totalPages, 
        Clubs = clubs 
    }; 
 
    return Ok(result); 
}
 
It's route is defined as ..api/clubs/{pageSize:int}/{pageNumber:int}/{orderBy:alpha?}. For example to return page 2 with a page size of 5 with the results ordered by the property 'name' the following url would be valid: http://localhost:<em>nnnnn</em>/api/clubs/5/2/name.




orderBy is optional if a value is provided, and the property name exists on the Club class it is used to build a property expression which is added to the clubQuery.
Client side (AngularJS)

The client application is in the app folder under the project root. It is a simple, single module SPA with two views to show the different approaches to paging.



A resource service called clubClientSvc is used by both the examples. It maps to the Clubs controller. The 'query' method has been configured to match the club controller action which supports paging.

JavaScript
angular 
    .module('app') 
    .factory('clubClientSvc', function ($resource) { 
        return $resource("api/clubs/:id", 
            { id: "@id" }, 
            {                   
                'query': { 
                    method: 'GET', 
                    url:'/api/clubs/:pageSize/:pageNumber/:orderBy', 
                    params: { pageSize: '@pageSize', pageNumber: '@pageNumber', orderBy: '@orderBy' } 
                }  
        }); 
    });
Simple paging

The simple paging approach provides a single list of items, which for this demo are football clubs to the user. If more clubs are available the user can query for the next page of items which are then added to the list.



The structure is straightforward with a controller (simpleCtrl), view and a service (simpleClubSvc) to abstract the paging logic out of the controller. 



The service exposes an array named clubs, which is the paged clubs list, a paging object of all the paging options and information and two functions load and clear.

JavaScript
service = { 
    load: load, 
    clear: clear, 
    clubs: [], 
    paging: { 
        options: angular.copy(initialOptions), 
        info: { 
            totalItems: 0, 
            totalPages: 1, 
            currentPage: 0, 
            sortableProperties: [ 
            "name", 
            "city" 
            ] 
        } 
    } 
}; 
 
service.paging.info.moreAvailable = function () { 
    return service.paging.info.currentPage < service.paging.info.totalPages; 
}
 
The load function calls the clubClientSvc query function defined earlier and once the resulting promise has been successfully resolved adds the returned page of clubs to the clubs array and increments the current page number.
JavaScript
function load() { 
    service.paging.info.currentPage += 1; 
 
    var queryArgs = { 
        pageSize: service.paging.options.size, 
        pageNumber: service.paging.info.currentPage, 
        orderBy: service.paging.options.orderBy 
    }; 
 
    return clubClientSvc.query(queryArgs).$promise.then( 
        function (result) {                     
            result.clubs.forEach(function (club) {                          
                service.clubs.push(club); 
            }); 
 
            service.paging.info.totalPages = result.totalPages; 
            service.paging.info.totalItems = result.totalCount; 
 
            return result.$promise; 
        }, function (result) { 
            service.paging.info.currentPage -= 1; 
            return $q.reject(result); 
        }); 
}
Full paging

The full paging approach gives the user a list of all the available pages which the user can then click on to view the clubs on that page. The page is loaded the first time it is clicked on.



The overall approach is similar with the addition of a directive called pager which handles the page navigation layout and logic.



The fullClubSvc exposes an array pages along with a paging object containing the paging information and options. 

JavaScript
service = { 
    initialize: initialize, 
    navigate: navigate, 
    clear: clear, 
    pages: [], 
    paging: { 
        options: angular.copy(initialOptions), 
        info: { 
            totalItems: 0, 
            totalPages: 1, 
            currentPage: 0, 
            sortableProperties: [ 
            "name", 
            "city" 
            ] 
        } 
    }                        
};
 It has an initialize function which makes the initial query to load the first page and set the paging info.

JavaScript
function initialize() { 
    var queryArgs = { 
        pageSize: service.paging.options.size, 
        pageNumber: service.paging.info.currentPage 
    }; 
 
    service.paging.info.currentPage = 1; 
 
    return clubClientSvc.query(queryArgs).$promise.then( 
        function (result) { 
            var newPage = { 
                number: pageNumber, 
                clubs: [] 
            }; 
            result.clubs.forEach(function (club) { 
                newPage.clubs.push(club); 
            }); 
 
            service.pages.push(newPage); 
            service.paging.info.currentPage = 1; 
            service.paging.info.totalPages = result.totalPages; 
 
            return result.$promise; 
        }, function (result) { 
            return $q.reject(result); 
        }); 
}
It also has a function navigate which is used by the pager directive to navigate between pages. The pager directive itself handles creating the list of pages from the total number of pages.

JavaScript
function navigate(pageNumber) { 
    var dfd = $q.defer(); 
 
    if (pageNumber > service.paging.info.totalPages) {                
        return dfd.reject({ error: "page number out of range" }); 
    } 
 
    if (service.pages[pageNumber]) { 
        service.paging.info.currentPage = pageNumber; 
        dfd.resolve(); 
    } else { 
        return load(pageNumber); 
    } 
 
    return dfd.promise; 
}
